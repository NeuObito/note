### django源码剖析之manage.py
当创建一个django项目时，django项目中会带有manage.py，这是django相关命令行的入口，比如python manage.py makemigrations等。接下来我们来看看执行这些命令的时候都发生了什么。manage.py如下所示：
```python
import os
import sys

if __name__ == "__main__":
    # 在系统环境中设置一个变量，其实就是在一个字典中设置了键值
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "***.settings")
    try:
        from django.core.management import execute_from_command_line
    except ImportError:
        # The above import may fail for some other reason. Ensure that the
        # issue is really that Django is missing to avoid masking other
        # exceptions on Python 2.
        try:
            import django
        except ImportError:
        # 没有安装django时抛出错误
            raise ImportError(
                "Couldn't import Django. Are you sure it's installed and "
                "available on your PYTHONPATH environment variable? Did you "
                "forget to activate a virtual environment?"
            )
        raise
    execute_from_command_line(sys.argv)
```
可以看出我们执行的命令(sys.argv包含了我们在cmd中输入的命令，为一个列表，以逗号隔开)被作为参数送给了execute_from_command_line函数。
execute_from_command_line函数：
```python
def execute_from_command_line(argv=None):
    """
    A simple method that runs a ManagementUtility.
    一个运行管理实用程序的方法。
    """
    utility = ManagementUtility(argv)
    utility.execute()
```
而execute_from_command_line函数里其实就是实例化了一个ManagementUtility，并运行了ManagementUtility实例的execute()方法。
```python
class ManagementUtility(object):
    """
    Encapsulates the logic of the django-admin and manage.py utilities.
    封装django-admin和manage.py实用程序的逻辑。
    """
    def __init__(self, argv=None):
        self.argv = argv or sys.argv[:]  # 获取命令
        self.prog_name = os.path.basename(self.argv[0])  # 获取命令的组件名，区分是django-admin还是manage.py
        self.settings_exception = None  # 记录异常

    def main_help_text(self, commands_only=False):
        """
        Returns the script's main help text, as a string.
        以字符串的形式返回脚本的主要帮助说明。
        """
        if commands_only:
            # get_commands()的作用是将命令名名称映射到它们的回调应用程序
            usage = sorted(get_commands().keys())
        else:
            usage = [
                "",
                "Type '%s help <subcommand>' for help on a specific subcommand." % self.prog_name,
                "",
                "Available subcommands:",
            ]
            commands_dict = defaultdict(lambda: [])  # 默认值为[]
            for name, app in six.iteritems(get_commands()):
                if app == 'django.core':
                    app = 'django'
                else:
                    app = app.rpartition('.')[-1]
                commands_dict[app].append(name)
            style = color_style()
            for app in sorted(commands_dict.keys()):
                usage.append("")
                usage.append(style.NOTICE("[%s]" % app))
                for name in sorted(commands_dict[app]):
                    usage.append("    %s" % name)
        ...
        return '\n'.join(usage)   
```
实例化的时候初始化了我们输入的参数，以及命令的前缀名，还有异常。最重要的是execute()方法：
```python
    def execute(self):
        """
        Given the command-line arguments, this figures out which subcommand is
        being run, creates a parser appropriate to that command, and runs it.
        """
        try:
            subcommand = self.argv[1]  # 获取子命令
        except IndexError:
            subcommand = 'help'  # Display help if no arguments were given.

        # Preprocess options to extract --settings and --pythonpath.
        # These options could affect the commands that are available, so they
        # must be processed early.
        parser = CommandParser(None, usage="%(prog)s subcommand [options] [args]", add_help=False)
        parser.add_argument('--settings')  # 每次执行命令，后台会自动添加这几个选项
        parser.add_argument('--pythonpath')
        parser.add_argument('args', nargs='*')  # catch-all
        try:
            """
            这里的optoins用于存储属性的简单对象，或者称之为填充好的命名空间。通过属性名称和值来判断相等性，并提供一个简单的字符串表示。
            类似于这样：Namespace(args=[], pythonpath=None, settings=None)
            args是一个列表：表示剩余的参数
            """
            options, args = parser.parse_known_args(self.argv[2:])
            """
            下面函数（handle_default_options）的主要作用为：
            包括了所有命令在这里应该接受的任何默认选项，以便ManagementUtility可以在搜索用户命令之前处理它们。
            """
            handle_default_options(options)
        except CommandError:
            pass  # Ignore any option errors at this point.

        try:
            settings.INSTALLED_APPS  # 检查apps配置是否正确，不正确则抛出错误
        except ImproperlyConfigured as exc:
            self.settings_exception = exc

        if settings.configured:
            # Start the auto-reloading dev server even if the code is broken.
            # The hardcoded condition is a code smell but we can't rely on a
            # flag on the command class because we haven't located it yet.
            if subcommand == 'runserver' and '--noreload' not in self.argv:
                # 如果子命令为runserver并且没有携带--noreload，则需 要检查一遍错误
                try:
                    autoreload.check_errors(django.setup)()
                except Exception:
                    # The exception will be raised later in the child process
                    # started by the autoreloader. Pretend it didn't happen by
                    # loading an empty list of applications.
                    apps.all_models = defaultdict(OrderedDict)
                    apps.app_configs = OrderedDict()
                    apps.apps_ready = apps.models_ready = apps.ready = True

                    # Remove options not compatible with the built-in runserver
                    # (e.g. options for the contrib.staticfiles' runserver).
                    # Changes here require manually testing as described in
                    # #27522.
                    _parser = self.fetch_command('runserver').create_parser('django', 'runserver')
                    _options, _args = _parser.parse_known_args(self.argv[2:])
                    for _arg in _args:
                        self.argv.remove(_arg)

            # In all other cases, django.setup() is required to succeed.
            else:
                """
                配置设置（这是访问第一个设置的副作用），配置日志记录并填充应用注册表。如果set_prefix为True，则设置线程局部urlresolvers脚本前缀，并填充每个App，加载每个app的配置和Models.
                """
                django.setup()

        self.autocomplete()

        if subcommand == 'help':
            if '--commands' in args:
                sys.stdout.write(self.main_help_text(commands_only=True) + '\n')
            elif len(options.args) < 1:
                sys.stdout.write(self.main_help_text() + '\n')
            else:
                self.fetch_command(options.args[0]).print_help(self.prog_name, options.args[0])
        # Special-cases: We want 'django-admin --version' and
        # 'django-admin --help' to work, for backwards compatibility.
        elif subcommand == 'version' or self.argv[1:] == ['--version']:
            sys.stdout.write(django.get_version() + '\n')
        elif self.argv[1:] in (['--help'], ['-h']):
            sys.stdout.write(self.main_help_text() + '\n')
        else:
            # 获取子命令的完整模块并通过完整模块执行命令
            self.fetch_command(subcommand).run_from_argv(self.argv)

```
我们的命令行被装进了CommandParser，该类其实是对ArgumentParser进行了一个符合django的继承和封装。命令行初始化完成之后，运行options, args = parser.parse_known_args(self.argv[2:])，该函数如下所示：
```python
    def parse_known_args(self, args=None, namespace=None):
        if args is None:
            # args default to the system args
            args = _sys.argv[1:]
        else:
            # make sure that args are mutable
            args = list(args)

        # default Namespace built from parser defaults
        if namespace is None:
            namespace = Namespace()

        # add any action defaults that aren't present
        for action in self._actions:
            if action.dest is not SUPPRESS:
                if not hasattr(namespace, action.dest):
                    if action.default is not SUPPRESS:
                        setattr(namespace, action.dest, action.default)

        # add any parser defaults that aren't present
        for dest in self._defaults:
            if not hasattr(namespace, dest):
                setattr(namespace, dest, self._defaults[dest])

        # parse the arguments and exit if there are any errors
        try:
            namespace, args = self._parse_known_args(args, namespace)
            if hasattr(namespace, _UNRECOGNIZED_ARGS_ATTR):
                args.extend(getattr(namespace, _UNRECOGNIZED_ARGS_ATTR))
                delattr(namespace, _UNRECOGNIZED_ARGS_ATTR)
            return namespace, args
        except ArgumentError:
            err = _sys.exc_info()[1]
            self.error(str(err))
```
由于该函数其实返回了一个namespace和args元组，那么这个Namespace到底是什么呢？
```python
class Namespace(_AttributeHolder):
    """Simple object for storing attributes.

    Implements equality by attribute names and values, and provides a simple
    string representation.
    用于存储属性的简单对象，或者称之为填充好的命名空间。通过属性名称和值来判断相等性，并提供一个简单的字符串表示。
    类似于这样：Namespace(args=[], pythonpath=None, settings=None)
    """

    def __init__(self, **kwargs):
        for name in kwargs:
            setattr(self, name, kwargs[name])

    __hash__ = None

    def __eq__(self, other):
        if not isinstance(other, Namespace):
            return NotImplemented
        return vars(self) == vars(other)

    def __ne__(self, other):
        if not isinstance(other, Namespace):
            return NotImplemented
        return not (self == other)

    def __contains__(self, key):
        return key in self.__dict__
```
Namespace初始化时，设置了名字和值。而其默认的值有解析器默认值构建。parse_known_args后面的几行都是进行一些默认值的设置。
返回命名空间和剩余参数之后，执行了handle_default_options(options):
```python
def handle_default_options(options):
    """
    Include any default options that all commands should accept here
    so that ManagementUtility can handle them before searching for
    user commands.
    """
    if options.settings:
        os.environ['DJANGO_SETTINGS_MODULE'] = options.settings
    if options.pythonpath:
        sys.path.insert(0, options.pythonpath)
```
handle_default_options将解析出来的options对象当做参数，判断settings和pythonpath是否存在，然后设置环境变量和python模块的搜索路径。
再之后是一些判断分支，判断是不是runserver，如果是则进行一些额外的检查。
然后是self.autocomplete()这个函数主要的功能是通过BASH去输出执行建议。
最后一个esle是重点，self.fetch_command(subcommand).run_from_argv(self.argv)再进行完默认设置以及排除完其余分支之后，才开始真正的执行这条命令。
fetch_command:
```python
def fetch_command(self, subcommand):
        """
        Tries to fetch the given subcommand, printing a message with the
        appropriate command called from the command line (usually
        "django-admin" or "manage.py") if it can't be found.
        """
        commands = get_commands()
        try:
            app_name = commands[subcommand]
        except KeyError:
            # This might trigger ImproperlyConfigured (masked in get_commands)
            settings.INSTALLED_APPS
            sys.stderr.write("Unknown command: %r\nType '%s help' for usage.\n" %
                (subcommand, self.prog_name))
            sys.exit(1)
        if isinstance(app_name, BaseCommand):
            # If the command is already loaded, use it directly.
            klass = app_name
        else:
            # 没有装载的命令通过下面这个函数进行封装后返回
            klass = load_command_class(app_name, subcommand)
        return klass
```
在Django源码中加入输出，get_commands返回的为所有命令：
```python
{'clearsessions': 'django.contrib.sessions', 'compilemessages': u'django.core',
'dumpdata': u'django.core', 'findstatic': 'django.contrib.staticfiles', 'loaddata': u'django.core', 'createcachetable': u'django.core', 'flush': u'django.core', 'squashmigrations': u'django.core', 'check': u'django.core', 'sqlmigrate': u'django.core', 'remove_stale_contenttypes': 'django.contrib.contenttypes', 'makemigrations': u'django.core', 'runserver': 'django.contrib.staticfiles', 'migrate':u'django.core', 'showmigrations': u'django.core', 'dbshell': u'django.core', 'changepassword': 'django.contrib.auth', 'test': u'django.core', 'sendtestemail': u'django.core', 'shell': u'django.core', 'sqlsequencereset': u'django.core', 'testserver': u'django.core', 'makemessages': u'django.core', 'startproject': u'django.core', 'collectstatic': 'django.contrib.staticfiles', 'diffsettings': u'django.core', 'inspectdb': u'django.core', 'startapp': u'django.core', 'createsuperuser': 'django.contrib.auth', 'sqlflush': u'django.core'}
```
通过这个字典，可以获得子命令对应的app_name。如果app_name是BaseCommand的一个实例，即已经装载，则直接使用。如果不是，则调用load_command_class(app_name, subcommand)函数进行装载。load_command_class(app_name, subcommand)：
```python
def load_command_class(app_name, name):
    """
    Given a command name and an application name, returns the Command
    class instance. All errors raised by the import process
    (ImportError, AttributeError) are allowed to propagate.
    """
    module = import_module('%s.management.commands.%s' % (app_name, name))
    return module.Command()
```
这个方法调用python中importlib库中的import_module方法将模块动态载入，然后返回载入模块的Command()。所以fetch_command函数返回的其实是一个Command对象，而所有的Command对象实际上都继承于BaseCommand。所以run_from_argv()方法实际是每个Command对BaseCommand的一个重写。
run_from_argv()：
```python
    def run_from_argv(self, argv):
        """
        Set up any environment changes requested (e.g., Python path
        and Django settings), then run this command. If the
        command raises a ``CommandError``, intercept it and print it sensibly
        to stderr. If the ``--traceback`` option is present or the raised
        ``Exception`` is not ``CommandError``, raise it.
        """
        self._called_from_command_line = True
        parser = self.create_parser(argv[0], argv[1])

        options = parser.parse_args(argv[2:])
        cmd_options = vars(options)
        # Move positional args out of options to mimic legacy optparse
        args = cmd_options.pop('args', ())
        handle_default_options(options)
        try:
            self.execute(*args, **cmd_options)
        except Exception as e:
            if options.traceback or not isinstance(e, CommandError):
                raise

            # SystemCheckError takes care of its own formatting.
            if isinstance(e, SystemCheckError):
                self.stderr.write(str(e), lambda x: x)
            else:
                self.stderr.write('%s: %s' % (e.__class__.__name__, e))
            sys.exit(1)
        finally:
            try:
                connections.close_all()
            except ImproperlyConfigured:
                # Ignore if connections aren't setup at this point (e.g. no
                # configured settings).
                pass
```
前面几行实际上也是CommandParser的一些设置。设置完成之后会运行self.execute():
```python
    def execute(self, *args, **options):
        """
        Try to execute this command, performing system checks if needed (as
        controlled by the ``requires_system_checks`` attribute, except if
        force-skipped).
        """
        if options['no_color']:
            self.style = no_style()
            self.stderr.style_func = None
        if options.get('stdout'):
            self.stdout = OutputWrapper(options['stdout'])
        if options.get('stderr'):
            self.stderr = OutputWrapper(options['stderr'], self.stderr.style_func)

        saved_locale = None
        if not self.leave_locale_alone:
            # Deactivate translations, because django-admin creates database
            # content like permissions, and those shouldn't contain any
            # translations.
            from django.utils import translation
            saved_locale = translation.get_language()
            translation.deactivate_all()

        try:
            if self.requires_system_checks and not options.get('skip_checks'):
                self.check()
            if self.requires_migrations_checks:
                self.check_migrations()
            output = self.handle(*args, **options)
            if output:
                if self.output_transaction:
                    connection = connections[options.get('database', DEFAULT_DB_ALIAS)]
                    output = '%s\n%s\n%s' % (
                        self.style.SQL_KEYWORD(connection.ops.start_transaction_sql()),
                        output,
                        self.style.SQL_KEYWORD(connection.ops.end_transaction_sql()),
                    )
                self.stdout.write(output)
        finally:
            if saved_locale is not None:
                translation.activate(saved_locale)
        return output
```
可以看到执行self.handle()之前会有两个判断，第一个判断为如果需要系统检测(默认为True)，则会进行一个检查；第二个为迁移检查(默认为False)，执行命令时会检查数据库和Models是否匹配。
self.handle():
```python
    def handle(self, *args, **options):
        """
        The actual logic of the command. Subclasses must implement
        this method.
        """
        raise NotImplementedError('subclasses of BaseCommand must provide a handle() method')
```
这是一个必须继承的函数，如果不继承就会抛出错误，所以返回的那些命令对象肯定会重写这个方法。再执行完handle之后，如果有返回，则会继续检查数据的可连接性。不可连接则报错，但是不会断开django。
我们来看一下runserver命令(django.core.management.commands.runserver.py，该命令继承了BaseCommand并重写了handle方法：
```python
class Command(BaseCommand):
    help = "Starts a lightweight Web server for development."
    requires_system_checks = False
    leave_locale_alone = True

    # 可以看到默认的端口和协议设置在这里
    default_port = '8000'
    protocol = 'http'
    server_cls = WSGIServer

    def handle(self, *args, **options):
        from django.conf import settings

        if not settings.DEBUG and not settings.ALLOWED_HOSTS:
            raise CommandError('You must set settings.ALLOWED_HOSTS if DEBUG is False.')

        self.use_ipv6 = options['use_ipv6']
        if self.use_ipv6 and not socket.has_ipv6:
            raise CommandError('Your Python does not support IPv6.')
        self._raw_ipv6 = False
        if not options['addrport']:
            self.addr = ''
            self.port = self.default_port
        else:
            m = re.match(naiveip_re, options['addrport'])
            if m is None:
                raise CommandError('"%s" is not a valid port number '
                                   'or address:port pair.' % options['addrport'])
            self.addr, _ipv4, _ipv6, _fqdn, self.port = m.groups()
            if not self.port.isdigit():
                raise CommandError("%r is not a valid port number." % self.port)
            if self.addr:
                if _ipv6:
                    self.addr = self.addr[1:-1]
                    self.use_ipv6 = True
                    self._raw_ipv6 = True
                elif self.use_ipv6 and not _fqdn:
                    raise CommandError('"%s" is not a valid IPv6 address.' % self.addr)
        if not self.addr:
            self.addr = '::1' if self.use_ipv6 else '127.0.0.1'
            self._raw_ipv6 = self.use_ipv6
        self.run(**options)
```
前面是一些参数设置，可以看到默认的地址为127.0.0.1。最后是执行，self.run():
```python
    def run(self, **options):
        """
        Runs the server, using the autoreloader if needed
        """
        use_reloader = options['use_reloader']

        if use_reloader:
            autoreload.main(self.inner_run, None, options)
        else:
            self.inner_run(None, **options)
```
这里又依赖了inner_run(None, **options):
```python
    def inner_run(self, *args, **options):
        # If an exception was silenced in ManagementUtility.execute in order
        # to be raised in the child process, raise it now.
        autoreload.raise_last_exception()

        threading = options['use_threading']
        # 'shutdown_message' is a stealth option.
        shutdown_message = options.get('shutdown_message', '')
        quit_command = 'CTRL-BREAK' if sys.platform == 'win32' else 'CONTROL-C'

        self.stdout.write("Performing system checks...\n\n")
        self.check(display_num_errors=True)
        # Need to check migrations here, so can't use the
        # requires_migrations_check attribute.
        self.check_migrations()
        now = datetime.now().strftime('%B %d, %Y - %X')
        if six.PY2:
            now = now.decode(get_system_encoding())
        self.stdout.write(now)
        self.stdout.write((
            "Django version %(version)s, using settings %(settings)r\n"
            "Starting development server at %(protocol)s://%(addr)s:%(port)s/\n"
            "Quit the server with %(quit_command)s.\n"
        ) % {
            "version": self.get_version(),
            "settings": settings.SETTINGS_MODULE,
            "protocol": self.protocol,
            "addr": '[%s]' % self.addr if self._raw_ipv6 else self.addr,
            "port": self.port,
            "quit_command": quit_command,
        })

        try:
            handler = self.get_handler(*args, **options)
            run(self.addr, int(self.port), handler,
                ipv6=self.use_ipv6, threading=threading, server_cls=self.server_cls)
        except socket.error as e:
            # Use helpful error messages instead of ugly tracebacks.
            ERRORS = {
                errno.EACCES: "You don't have permission to access that port.",
                errno.EADDRINUSE: "That port is already in use.",
                errno.EADDRNOTAVAIL: "That IP address can't be assigned to.",
            }
            try:
                error_text = ERRORS[e.errno]
            except KeyError:
                error_text = force_text(e)
            self.stderr.write("Error: %s" % error_text)
            # Need to use an OS exit because sys.exit doesn't work in a thread
            os._exit(1)
        except KeyboardInterrupt:
            if shutdown_message:
                self.stdout.write(shutdown_message)
            sys.exit(0)

```
而这里最重要的是self.get_handler:
```python
    def get_handler(self, *args, **options):
        """
        Returns the default WSGI handler for the runner.
        """
        return get_internal_wsgi_application()
```
该函数返回了一个默认的WSGI Handler。
get_internal_wsgi_application():
```python
def get_internal_wsgi_application():
    """
    Loads and returns the WSGI application as configured by the user in
    ``settings.WSGI_APPLICATION``. With the default ``startproject`` layout,
    this will be the ``application`` object in ``projectname/wsgi.py``.

    This function, and the ``WSGI_APPLICATION`` setting itself, are only useful
    for Django's internal server (runserver); external WSGI servers should just
    be configured to point to the correct application object directly.

    If settings.WSGI_APPLICATION is not set (is ``None``), we just return
    whatever ``django.core.wsgi.get_wsgi_application`` returns.
    """
    from django.conf import settings
    app_path = getattr(settings, 'WSGI_APPLICATION')
    if app_path is None:
        return get_wsgi_application()

    try:
        return import_string(app_path)
    except ImportError as e:
        msg = (
            "WSGI application '%(app_path)s' could not be loaded; "
            "Error importing module: '%(exception)s'" % ({
                'app_path': app_path,
                'exception': e,
            })
        )
        six.reraise(ImproperlyConfigured, ImproperlyConfigured(msg),
                    sys.exc_info()[2])
```
该函数又调用了get_wsgi_application:
```python
def get_wsgi_application():
    """
    The public interface to Django's WSGI support. Should return a WSGI
    callable.

    Allows us to avoid making django.core.handlers.WSGIHandler public API, in
    case the internal WSGI implementation changes or moves in the future.
    """
    django.setup(set_prefix=False)
    return WSGIHandler()
```
```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super(WSGIHandler, self).__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = [(str(k), str(v)) for k, v in response.items()]
        for c in response.cookies.values():
            response_headers.append((str('Set-Cookie'), str(c.output(header=''))))
        start_response(force_str(status), response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```
可以看出，在执行runserver时，都做了什么操作，WSGIHandler初始化的时候，加载的应用的中间件，并发送了请求开始信号，最后返回了一个response。