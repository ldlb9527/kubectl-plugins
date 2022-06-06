![](https://img.shields.io/badge/kubectl-plugin-green)
# 使用插件扩展kubectl
*如果你不关注源码的实现可直接前往第三部分-[3.扩展kubectl](#three)
## 1.什么是kubectl plugin?
* `kubectl`是kubernetes官方基于cobra开发的用于集群管理的命令行工具。
* 从 1.12 版本开始，`kubectl`包含一个插件机制，允许您使用自定义插件扩展 `kubectl`。
## 2.kubectl plugin的源码分析
``` go
func main() {
    //创建kubectl命令的默认参数
	command := cmd.NewDefaultKubectlCommand() 
	//执行command
	if err := cli.RunNoErrOutput(command); err != nil {
		// Pretty-print the error and exit with an error.
		util.CheckErr(err)
	}
```
* kubectl的入口函数位于`cmd/kubectl/kubectl.go`，核心函数在于`NewDefaultKubectlCommand()`
***
``` go
// NewDefaultKubectlCommand creates the `kubectl` command with default arguments
func NewDefaultKubectlCommand() *cobra.Command {
	return NewDefaultKubectlCommandWithArgs(KubectlOptions{
		PluginHandler: NewDefaultPluginHandler(plugin.ValidPluginFilenamePrefixes),
		Arguments:     os.Args,
		ConfigFlags:   defaultConfigFlags,
		IOStreams:     genericclioptions.IOStreams{In: os.Stdin, Out: os.Stdout, ErrOut: os.Stderr},
	})
}
```
* 主要逻辑为初始化`PluginHandler`,`Arguments`, `ConfigFlags`, `IOStreams` 字段,构造`KubectlOptions`对象,作为`NewDefaultKubectlCommandWithArgs`的入参
***
``` go
// NewDefaultKubectlCommandWithArgs creates the `kubectl` command with arguments
func NewDefaultKubectlCommandWithArgs(o KubectlOptions) *cobra.Command {
    //创建一个kubectl命令
	cmd := NewKubectlCommand(o)

	if o.PluginHandler == nil {
		return cmd
	}

    //o.Arguments为标准输入，例如 kubetl apply -f test.yaml
	if len(o.Arguments) > 1 { 
	    //取kubectl后的参数，例如 apply -f test.yaml
		cmdPathPieces := o.Arguments[1:] 

		// only look for suitable extension executables if
		// the specified command does not already exist
		//通过Find去匹配命令，如果命令不存在，则寻找合适的扩展可执行文件
		if _, _, err := cmd.Find(cmdPathPieces); err != nil {
			// Also check the commands that will be added by Cobra.
			// These commands are only added once rootCmd.Execute() is called, so we
			// need to check them explicitly here.
			var cmdName string // first "non-flag" arguments
			for _, arg := range cmdPathPieces {
				if !strings.HasPrefix(arg, "-") {
					cmdName = arg
					break
				}
			}

			switch cmdName {
			case "help", cobra.ShellCompRequestCmd, cobra.ShellCompNoDescRequestCmd:
				// Don't search for a plugin
			default:
				if err := HandlePluginCommand(o.PluginHandler, cmdPathPieces); err != nil {
					fmt.Fprintf(o.IOStreams.ErrOut, "Error: %v\n", err)
					os.Exit(1)
				}
			}
		}
	}

	return cmd
}
```
* 重点在`NewKubectlCommand`创建一个kubectl和`HandlePluginCommand`处理插件命令
***
``` go
// NewKubectlCommand creates the `kubectl` command and its nested children.
func NewKubectlCommand(o KubectlOptions) *cobra.Command {
	warningHandler := rest.NewWarningWriter(o.IOStreams.ErrOut, rest.WarningWriterOptions{Deduplicate: true, Color: term.AllowsColorOutput(o.IOStreams.ErrOut)})
	warningsAsErrors := false
	// Parent command to which all subcommands are added.
	//创建主命令
	cmds := &cobra.Command{
		Use:   "kubectl",
		Short: i18n.T("kubectl controls the Kubernetes cluster manager"),
		Long: templates.LongDesc(`
      kubectl controls the Kubernetes cluster manager.

      Find more information at:
            https://kubernetes.io/docs/reference/kubectl/`),
		Run: runHelp,
		// Hook before and after Run initialize and write profiles to disk,
		// respectively.
		//初始化后，在运行指令前的钩子
		PersistentPreRunE: func(*cobra.Command, []string) error {
			rest.SetDefaultWarningHandler(warningHandler)
			//这里是做pprof性能分析，跳转到对应代码可以看到，我们可以用参数 --profile xxx 来采集性能指标，默认保存在当前目录下的profile.pprof中
			return initProfiling()
		},
		//运行指令后的钩子
		PersistentPostRunE: func(*cobra.Command, []string) error {
			if err := flushProfiling(); err != nil {
				return err
			}
			if warningsAsErrors {
				count := warningHandler.WarningCount()
				switch count {
				case 0:
					// no warnings
				case 1:
					return fmt.Errorf("%d warning received", count)
				default:
					return fmt.Errorf("%d warnings received", count)
				}
			}
			return nil
		},
	}
	// From this point and forward we get warnings on flags that contain "_" separators
	// when adding them with hyphen instead of the original name.
	//初始化设置cmds命令
	cmds.SetGlobalNormalizationFunc(cliflag.WarnWordSepNormalizeFunc)

	flags := cmds.PersistentFlags()

	addProfilingFlags(flags)

	flags.BoolVar(&warningsAsErrors, "warnings-as-errors", warningsAsErrors, "Treat warnings received from the server as errors and exit with a non-zero exit code")

	kubeConfigFlags := o.ConfigFlags
	if kubeConfigFlags == nil {
		kubeConfigFlags = defaultConfigFlags
	}
	kubeConfigFlags.AddFlags(flags)
	matchVersionKubeConfigFlags := cmdutil.NewMatchVersionFlags(kubeConfigFlags)
	matchVersionKubeConfigFlags.AddFlags(flags)
	// Updates hooks to add kubectl command headers: SIG CLI KEP 859.
	addCmdHeaderHooks(cmds, kubeConfigFlags)

    //初始化一个工厂
	f := cmdutil.NewFactory(matchVersionKubeConfigFlags)

	// Sending in 'nil' for the getLanguageFn() results in using
	// the LANG environment variable.
	//
	// TODO: Consider adding a flag or file preference for setting
	// the language, instead of just loading from the LANG env. variable.
	i18n.LoadTranslations("kubectl", nil)

	// Proxy command is incompatible with CommandHeaderRoundTripper, so
	// clear the WrapConfigFn before running proxy command.
	proxyCmd := proxy.NewCmdProxy(f, o.IOStreams)
	proxyCmd.PreRun = func(cmd *cobra.Command, args []string) {
		kubeConfigFlags.WrapConfigFn = nil
	}

	// Avoid import cycle by setting ValidArgsFunction here instead of in NewCmdGet()
	getCmd := get.NewCmdGet("kubectl", f, o.IOStreams)
	getCmd.ValidArgsFunction = utilcomp.ResourceTypeAndNameCompletionFunc(f)
    //kubectl定义了7类命令，Message为分类标签
	groups := templates.CommandGroups{
		{
			Message: "Basic Commands (Beginner):",
			Commands: []*cobra.Command{
				create.NewCmdCreate(f, o.IOStreams),
				expose.NewCmdExposeService(f, o.IOStreams),
				run.NewCmdRun(f, o.IOStreams),
				set.NewCmdSet(f, o.IOStreams),
			},
		},
		{
			Message: "Basic Commands (Intermediate):",
			Commands: []*cobra.Command{
				explain.NewCmdExplain("kubectl", f, o.IOStreams),
				getCmd,
				edit.NewCmdEdit(f, o.IOStreams),
				delete.NewCmdDelete(f, o.IOStreams),
			},
		},
		{
			Message: "Deploy Commands:",
			Commands: []*cobra.Command{
				rollout.NewCmdRollout(f, o.IOStreams),
				scale.NewCmdScale(f, o.IOStreams),
				autoscale.NewCmdAutoscale(f, o.IOStreams),
			},
		},
		{
			Message: "Cluster Management Commands:",
			Commands: []*cobra.Command{
				certificates.NewCmdCertificate(f, o.IOStreams),
				clusterinfo.NewCmdClusterInfo(f, o.IOStreams),
				top.NewCmdTop(f, o.IOStreams),
				drain.NewCmdCordon(f, o.IOStreams),
				drain.NewCmdUncordon(f, o.IOStreams),
				drain.NewCmdDrain(f, o.IOStreams),
				taint.NewCmdTaint(f, o.IOStreams),
			},
		},
		{
			Message: "Troubleshooting and Debugging Commands:",
			Commands: []*cobra.Command{
				describe.NewCmdDescribe("kubectl", f, o.IOStreams),
				logs.NewCmdLogs(f, o.IOStreams),
				attach.NewCmdAttach(f, o.IOStreams),
				cmdexec.NewCmdExec(f, o.IOStreams),
				portforward.NewCmdPortForward(f, o.IOStreams),
				proxyCmd,
				cp.NewCmdCp(f, o.IOStreams),
				auth.NewCmdAuth(f, o.IOStreams),
				debug.NewCmdDebug(f, o.IOStreams),
			},
		},
		{
			Message: "Advanced Commands:",
			Commands: []*cobra.Command{
				diff.NewCmdDiff(f, o.IOStreams),
				apply.NewCmdApply("kubectl", f, o.IOStreams),
				patch.NewCmdPatch(f, o.IOStreams),
				replace.NewCmdReplace(f, o.IOStreams),
				wait.NewCmdWait(f, o.IOStreams),
				kustomize.NewCmdKustomize(o.IOStreams),
			},
		},
		{
			Message: "Settings Commands:",
			Commands: []*cobra.Command{
				label.NewCmdLabel(f, o.IOStreams),
				annotate.NewCmdAnnotate("kubectl", f, o.IOStreams),
				completion.NewCmdCompletion(o.IOStreams.Out, ""),
			},
		},
	}
	groups.Add(cmds)

	filters := []string{"options"}

	// Hide the "alpha" subcommand if there are no alpha commands in this build.
	//如果在这个版本中没有alpha命令，隐藏“alpha”子命令
	alpha := NewCmdAlpha(f, o.IOStreams)
	if !alpha.HasSubCommands() {
		filters = append(filters, alpha.Name())
	}

	templates.ActsAsRootCommand(cmds, filters, groups...)

	utilcomp.SetFactoryForCompletion(f)
	registerCompletionFuncForGlobalFlags(cmds, f)

    //添加其余子命令，包括alpha/config/plugin/version/api-versions/api-resources/options
	cmds.AddCommand(alpha)
	cmds.AddCommand(cmdconfig.NewCmdConfig(clientcmd.NewDefaultPathOptions(), o.IOStreams))
	// 增加plugin子命令
	cmds.AddCommand(plugin.NewCmdPlugin(o.IOStreams))
	cmds.AddCommand(version.NewCmdVersion(f, o.IOStreams))
	cmds.AddCommand(apiresources.NewCmdAPIVersions(f, o.IOStreams))
	cmds.AddCommand(apiresources.NewCmdAPIResources(f, o.IOStreams))
	cmds.AddCommand(options.NewCmdOptions(o.IOStreams.Out))

	// Stop warning about normalization of flags. That makes it possible to
	// add the klog flags later.
	cmds.SetGlobalNormalizationFunc(cliflag.WordSepNormalizeFunc)
	return cmds
}
```
* 7类命令中有不同实现，选择基础命令的`NewCmdCreate`进行查看
***
``` go
// NewCmdCreate returns new initialized instance of create sub command
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
    //create子命令的相关选项
	o := NewCreateOptions(ioStreams)

    //create子命令
	cmd := &cobra.Command{
		Use:                   "create -f FILENAME",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Create a resource from a file or from stdin"),
		Long:                  createLong,
		Example:               createExample,
		Run: func(cmd *cobra.Command, args []string) {
		    //验证参数
			if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
				ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
				defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
				defaultRunFunc(cmd, args)
				return
			}
			cmdutil.CheckErr(o.Complete(f, cmd, args))
			cmdutil.CheckErr(o.Validate())
			//执行逻辑是在这里的RunCreate
			cmdutil.CheckErr(o.RunCreate(f, cmd))
		},
	}

	// bind flag structs
	o.RecordFlags.AddFlags(cmd)

	usage := "to use to create the resource"
	//加入文件名选项的flag -f，保存到o.FilenameOptions.Filenames中，对应上面
	cmdutil.AddFilenameOptionFlags(cmd, &o.FilenameOptions, usage)
	cmdutil.AddValidateFlags(cmd)
	cmd.Flags().BoolVar(&o.EditBeforeCreate, "edit", o.EditBeforeCreate, "Edit the API resource before creating")
	cmd.Flags().Bool("windows-line-endings", runtime.GOOS == "windows",
		"Only relevant if --edit=true. Defaults to the line ending native to your platform.")
	cmdutil.AddApplyAnnotationFlags(cmd)
	cmdutil.AddDryRunFlag(cmd)
	cmdutil.AddLabelSelectorFlagVar(cmd, &o.Selector)
	cmd.Flags().StringVar(&o.Raw, "raw", o.Raw, "Raw URI to POST to the server.  Uses the transport specified by the kubeconfig file.")
	cmdutil.AddFieldManagerFlagVar(cmd, &o.fieldManager, "kubectl-create")

	o.PrintFlags.AddFlags(cmd)

	// create subcommands
	//create的子命令
	cmd.AddCommand(NewCmdCreateNamespace(f, ioStreams))
	cmd.AddCommand(NewCmdCreateQuota(f, ioStreams))
	cmd.AddCommand(NewCmdCreateSecret(f, ioStreams))
	cmd.AddCommand(NewCmdCreateConfigMap(f, ioStreams))
	cmd.AddCommand(NewCmdCreateServiceAccount(f, ioStreams))
	cmd.AddCommand(NewCmdCreateService(f, ioStreams))
	cmd.AddCommand(NewCmdCreateDeployment(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateClusterRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRole(f, ioStreams))
	cmd.AddCommand(NewCmdCreateRoleBinding(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePodDisruptionBudget(f, ioStreams))
	cmd.AddCommand(NewCmdCreatePriorityClass(f, ioStreams))
	cmd.AddCommand(NewCmdCreateJob(f, ioStreams))
	cmd.AddCommand(NewCmdCreateCronJob(f, ioStreams))
	cmd.AddCommand(NewCmdCreateIngress(f, ioStreams))
	cmd.AddCommand(NewCmdCreateToken(f, ioStreams))
	return cmd
}
```
* 首先初始化create子命令的相关选项,其次验证参数并运行`RunCreate`,然后create的子命令
***
``` go
// RunCreate performs the creation
func (o *CreateOptions) RunCreate(f cmdutil.Factory, cmd *cobra.Command) error {
     // f为传入的Factory，主要是封装了与kube-apiserver交互客户端
	// raw only makes sense for a single file resource multiple objects aren't likely to do what you want.
	// the validator enforces this, so
	if len(o.Raw) > 0 {
		restClient, err := f.RESTClient()
		if err != nil {
			return err
		}
		return rawhttp.RawPost(restClient, o.IOStreams, o.Raw, o.FilenameOptions.Filenames[0])
	}

	if o.EditBeforeCreate {
		return RunEditOnCreate(f, o.PrintFlags, o.RecordFlags, o.IOStreams, cmd, &o.FilenameOptions, o.fieldManager)
	}

	schema, err := f.Validator(o.ValidationDirective, o.FieldValidationVerifier)
	if err != nil {
		return err
	}

	cmdNamespace, enforceNamespace, err := f.ToRawKubeConfigLoader().Namespace()
	if err != nil {
		return err
	}

    //实例化Builder
	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		//读取文件信息，发现除了支持简单的本地文件，也支持标准输入和http/https协议访问的文件，保存为Visitor
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
	err = r.Err()
	if err != nil {
		return err
	}

	count := 0
	//调用visit函数，创建资源
	err = r.Visit(func(info *resource.Info, err error) error {
		if err != nil {
			return err
		}
		if err := util.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, scheme.DefaultJSONEncoder()); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}

		if err := o.Recorder.Record(info.Object); err != nil {
			klog.V(4).Infof("error recording current command: %v", err)
		}

		if o.DryRunStrategy != cmdutil.DryRunClient {
			if o.DryRunStrategy == cmdutil.DryRunServer {
				if err := o.DryRunVerifier.HasSupport(info.Mapping.GroupVersionKind); err != nil {
					return cmdutil.AddSourceToErr("creating", info.Source, err)
				}
			}
			obj, err := resource.
				NewHelper(info.Client, info.Mapping).
				DryRun(o.DryRunStrategy == cmdutil.DryRunServer).
				WithFieldManager(o.fieldManager).
				WithFieldValidation(o.ValidationDirective).
				Create(info.Namespace, true, info.Object)
			if err != nil {
				return cmdutil.AddSourceToErr("creating", info.Source, err)
			}
			info.Refresh(obj, true)
		}

		count++

		return o.PrintObj(info.Object)
	})
	if err != nil {
		return err
	}
	if count == 0 {
		return fmt.Errorf("no objects passed to create")
	}
	return nil
}
```
* RunCreate的主要逻辑为初始化工厂类，初始化builder，利用观察者模式处理资源创建 封装client 和object 发送给apiserver，细节不在往下查看
* 具体关注`NewDefaultKubectlCommandWithArgs`函数处理plugin的逻辑
***
``` go
if len(o.Arguments) > 1 {
		cmdPathPieces := o.Arguments[1:]

		// only look for suitable extension executables if
		// the specified command does not already exist
		if _, _, err := cmd.Find(cmdPathPieces); err != nil {
			// Also check the commands that will be added by Cobra.
			// These commands are only added once rootCmd.Execute() is called, so we
			// need to check them explicitly here.
			var cmdName string // first "non-flag" arguments
			for _, arg := range cmdPathPieces {
				if !strings.HasPrefix(arg, "-") {
					cmdName = arg
					break
				}
			}

			switch cmdName {
			case "help", cobra.ShellCompRequestCmd, cobra.ShellCompNoDescRequestCmd:
				// Don't search for a plugin
			default:
				if err := HandlePluginCommand(o.PluginHandler, cmdPathPieces); err != nil {
					fmt.Fprintf(o.IOStreams.ErrOut, "Error: %v\n", err)
					os.Exit(1)
				}
			}
		}
	}
```
* 通过`o.Arguments`判断是否执行plugin,判定执行后会调用`HandlePluginCommand`,具体逻辑为：
* `o.Arguments`>1
* command未在cmds中注册(Find函数中找不到)
* command的名称不是 help, __complete, __completeNoDesc
***
* `kubectl plugin`支持查找和执行功能
* `kubectl plugin list`由kubectl原生支持，在上文函数`NewKubectlCommand`有一个`plugin.NewCmdPlugin(o.IOStreams)`
``` go
func NewCmdPlugin(streams genericclioptions.IOStreams) *cobra.Command {
	cmd := &cobra.Command{
		Use:                   "plugin [flags]",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Provides utilities for interacting with plugins"),
		Long:                  pluginLong,
		Run: func(cmd *cobra.Command, args []string) {
			cmdutil.DefaultSubCommandRun(streams.ErrOut)(cmd, args)
		},
	}

	cmd.AddCommand(NewCmdPluginList(streams)) // li
	return cmd
}
```
* `NewCmdPlugin`会新建一个plugin的cmd，然后追加`NewCmdPluginList`子命令,`kubectl plugin list`功能就在NewCmdPluginList中实现
``` go
// NewCmdPluginList provides a way to list all plugin executables visible to kubectl
func NewCmdPluginList(streams genericclioptions.IOStreams) *cobra.Command {
	o := &PluginListOptions{
		IOStreams: streams,
	}

	cmd := &cobra.Command{
		Use:     "list",
		Short:   i18n.T("List all visible plugin executables on a user's PATH"),
		Example: pluginExample,
		Long:    pluginListLong,
		Run: func(cmd *cobra.Command, args []string) {
			cmdutil.CheckErr(o.Complete(cmd))
			cmdutil.CheckErr(o.Run())
		},
	}

	cmd.Flags().BoolVar(&o.NameOnly, "name-only", o.NameOnly, "If true, display only the binary name of each plugin, rather than its full path")
	return cmd
}
```
* `NewCmdPluginList`的主要逻辑为构造`PluginListOptions`,`PluginListOptions`绑定有两个方法:`Complete`和`Run`
***
``` go
func (o *PluginListOptions) Complete(cmd *cobra.Command) error {
	o.Verifier = &CommandOverrideVerifier{
		root:        cmd.Root(),
		seenPlugins: make(map[string]string),
	}

	o.PluginPaths = filepath.SplitList(os.Getenv("PATH"))
	return nil
}
```
* `Verifier`检验是否为可执行文件,是否可能被其他plugin覆盖,是否被原生命令行覆盖,`PluginPaths`为执行路径`Run`函数会遍历PluginPaths，去寻找符合plugin要求的文件
***
``` go
func (o *PluginListOptions) Run() error {
	pluginsFound := false
	isFirstFile := true
	pluginErrors := []error{}
	pluginWarnings := 0

	for _, dir := range uniquePathsList(o.PluginPaths) {
		if len(strings.TrimSpace(dir)) == 0 {
			continue
		}

		files, err := ioutil.ReadDir(dir)
		if err != nil {
			if _, ok := err.(*os.PathError); ok {
				fmt.Fprintf(o.ErrOut, "Unable to read directory %q from your PATH: %v. Skipping...\n", dir, err)
				continue
			}

			pluginErrors = append(pluginErrors, fmt.Errorf("error: unable to read directory %q in your PATH: %v", dir, err))
			continue
		}

		for _, f := range files {
			if f.IsDir() {
				continue
			}
			if !hasValidPrefix(f.Name(), ValidPluginFilenamePrefixes) {
				continue
			}

			if isFirstFile {
				fmt.Fprintf(o.Out, "The following compatible plugins are available:\n\n")
				pluginsFound = true
				isFirstFile = false
			}

			pluginPath := f.Name()
			if !o.NameOnly {
				pluginPath = filepath.Join(dir, pluginPath)
			}

			fmt.Fprintf(o.Out, "%s\n", pluginPath)
			if errs := o.Verifier.Verify(filepath.Join(dir, f.Name())); len(errs) != 0 {
				for _, err := range errs {
					fmt.Fprintf(o.ErrOut, "  - %s\n", err)
					pluginWarnings++
				}
			}
		}
	}

	if !pluginsFound {
		pluginErrors = append(pluginErrors, fmt.Errorf("error: unable to find any kubectl plugins in your PATH"))
	}

	if pluginWarnings > 0 {
		if pluginWarnings == 1 {
			pluginErrors = append(pluginErrors, fmt.Errorf("error: one plugin warning was found"))
		} else {
			pluginErrors = append(pluginErrors, fmt.Errorf("error: %v plugin warnings were found", pluginWarnings))
		}
	}
	if len(pluginErrors) > 0 {
		errs := bytes.NewBuffer(nil)
		for _, e := range pluginErrors {
			fmt.Fprintln(errs, e)
		}
		return fmt.Errorf("%s", errs.String())
	}

	return nil
}
```
* `Run`函数的主要逻辑为：
* 遍历o.PluginPaths，读目录下全部文件
* 判断是否为文件
* 判断是否为plugin文件，以 kubectl- 开头
* 找到第一个plugin文件时，写入The following compatible plugins are available
* 将 plugin 文件写入标准输出
* 判断是否可执行，如果不是则将信息写入错误输出
***
``` go
// HandlePluginCommand receives a pluginHandler and command-line arguments and attempts to find
// a plugin executable on the PATH that satisfies the given arguments.
func HandlePluginCommand(pluginHandler PluginHandler, cmdArgs []string) error {
	var remainingArgs []string // all "non-flag" arguments
	for _, arg := range cmdArgs {
		if strings.HasPrefix(arg, "-") {
			break
		}
		remainingArgs = append(remainingArgs, strings.Replace(arg, "-", "_", -1))
	}

	if len(remainingArgs) == 0 {
		// the length of cmdArgs is at least 1
		return fmt.Errorf("flags cannot be placed before plugin name: %s", cmdArgs[0])
	}

	foundBinaryPath := ""

	// attempt to find binary, starting at longest possible name with given cmdArgs
	for len(remainingArgs) > 0 {
	    //获取插件全路径
		path, found := pluginHandler.Lookup(strings.Join(remainingArgs, "-"))
		if !found {
			remainingArgs = remainingArgs[:len(remainingArgs)-1]
			continue
		}

		foundBinaryPath = path
		break
	}

	if len(foundBinaryPath) == 0 {
		return nil
	}

	// invoke cmd binary relaying the current environment and args given
	//执行
	if err := pluginHandler.Execute(foundBinaryPath, cmdArgs[len(remainingArgs):], os.Environ()); err != nil {
		return err
	}

	return nil
}
```
* 判定执行插件命令后，调用`HandlePluginCommand`,该函数接受`pluginHandler PluginHandler, cmdArgs []string`
* `pluginHandler`实现了两个方法:`Lookup`和`Execute`
***
``` go
// Lookup implements PluginHandler
func (h *DefaultPluginHandler) Lookup(filename string) (string, bool) {
	for _, prefix := range h.ValidPrefixes {
		path, err := exec.LookPath(fmt.Sprintf("%s-%s", prefix, filename))
		if err != nil || len(path) == 0 {
			continue
		}
		return path, true
	}

	return "", false
}

// Execute implements PluginHandler
func (h *DefaultPluginHandler) Execute(executablePath string, cmdArgs, environment []string) error {

	// Windows does not support exec syscall.
	if runtime.GOOS == "windows" {
		cmd := exec.Command(executablePath, cmdArgs...)
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
		cmd.Stdin = os.Stdin
		cmd.Env = environment
		err := cmd.Run()
		if err == nil {
			os.Exit(0)
		}
		return err
	}

	// invoke cmd binary relaying the environment and args given
	// append executablePath to cmdArgs, as execve will make first argument the "binary name".
	return syscall.Exec(executablePath, append([]string{executablePath}, cmdArgs...), environment)
}
```
* `Lookup`: 在PATH中查找满足条件的可执行文件,Execute:调用`syscall.Exec`执行
***
* 了解更多扩展`kubectl`请查看[官方文档-extend-kubectl](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)
***
## <a id="three"></a>3.扩展kubectl
