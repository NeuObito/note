### authenticate函数（django.contrib.auth)
这个函数首先要获取到后台验证方式：
```python
def authenticate(request=None, **credentials):
    """
    If the given credentials are valid, return a User object.
    """
    for backend, backend_path in _get_backends(return_tuples=True):
        try:
            user = _authenticate_with_backend(backend, backend_path, request, credentials)
        except PermissionDenied:
            # This backend says to stop in our tracks - this user should not be allowed in at all.
            break
        if user is None:
            continue
        # Annotate the user object with the path of the backend.
        user.backend = backend_path
        return user

    # The credentials supplied are invalid to all backends, fire signal
    user_login_failed.send(sender=__name__, credentials=_clean_credentials(credentials), request=request)
```
如果用户有自己的需求，如使用邮箱登录等，可以写自己的认证方式，具体做法是在你认证的app中新建一个backend.py文件，文件里定义自己的认证方式。如果你想使用邮箱登录，则可以使用如下写法：
```python
class EmailAuthBackend(object):
    def authenticate(self, username=None, password=None):
        try:
            user = User.objects.get(email=username)
            if user.check_password(password):
                return user
            return None
        except User.DoesNotExist:
            return None

    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None

```
然后在settings.py文件中加入：
```python
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'account.authentication.EmailAuthBackend',
)
```
如果没有自己定义，django会给出一个默认的值：
```python
def _get_backends(return_tuples=False):
    backends = []
    for backend_path in settings.AUTHENTICATION_BACKENDS:
        backend = load_backend(backend_path)
        backends.append((backend, backend_path) if return_tuples else backend)
    if not backends:
        raise ImproperlyConfigured(
            'No authentication backends have been defined. Does '
            'AUTHENTICATION_BACKENDS contain anything?'
        )
    return backends
```
在settings类中(djang.conf.__init__.py)，初始化时先会传入默认值：
```python
settings = LazySettings()

class LazySettings(LazyObject):
    # ....
    def configure(self, default_settings=global_settings, **options):
        """
        Called to manually configure the settings. The 'default_settings'
        parameter sets where to retrieve any unspecified values from (its
        argument must support attribute access (__getattr__)).
        """
        if self._wrapped is not empty:
            raise RuntimeError('Settings already configured.')
        holder = UserSettingsHolder(default_settings)
        for name, value in options.items():
            setattr(holder, name, value)
        self._wrapped = holder

```
由此可见默认值在一个叫global_settings的文件中，打开可以找到默认的后端验证方式：
```python
AUTHENTICATION_BACKENDS = ['django.contrib.auth.backends.ModelBackend']
```
所有如果你没有定义验证方式，而且还删除了默认值，那么就会抛出一个找不到验证方式的错误。
检查完后端验证方式之后，django会做函数验证：
```python 
inspect.getcallargs(backend.authenticate, request=request, **credentials)
```
函数验证通过，就使用验证方式进行验证：
```python
backend.authenticate(*args, **credentials)
```
既然默认的是django.contrib.auth.backends.ModelBackend，那么就进去探个究竟。
```python
class ModelBackend(object):
    """
    Authenticates against settings.AUTH_USER_MODEL.
    """

    def authenticate(self, request, username=None, password=None, **kwargs):
        if username is None:
            username = kwargs.get(UserModel.USERNAME_FIELD)
        try:
            user = UserModel._default_manager.get_by_natural_key(username)
        except UserModel.DoesNotExist:
            # Run the default password hasher once to reduce the timing
            # difference between an existing and a non-existing user (#20760).
            UserModel().set_password(password)
        else:
            if user.check_password(password) and self.user_can_authenticate(user):
                return user

    def user_can_authenticate(self, user):
        """
        Reject users with is_active=False. Custom user models that don't have
        that attribute are allowed.
        """
        is_active = getattr(user, 'is_active', None)
        return is_active or is_active is None

```
无论是自带的还是默认的，必须都有authenticate函数。该函数首先验证是否有用户名，如果没有携带用户名，就去UserModel中取USERNAME_FIELD字段。如果我们自己重写了User类，无论是通过何种方式，都可以在重写的用户类中写一个USERNAME_FILED字段，该字段表示你要通过该字段登录。django默认的用户类User，继承了AbstratctUser类，该类中定义了USERNAME_FIELD字段，默认为username。
```python
USERNAME_FIELD = 'username'
```
获取到用户名之后，django会去数据库中拿到user，之后再验证密码是否正确，如果密码正确并且user.is_active是True，则返回user(自定义用户结构可以没有is_active属性)。如果不能通过用户名获取到用户，django会运行一次密码哈希函数，来减少存在用户和不存在用户之间的差异。
