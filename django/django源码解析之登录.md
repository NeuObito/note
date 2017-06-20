### login函数(django.contrib.auth.__init__)
该函数整体如下所示：
```python
SESSION_KEY = '_auth_user_id'

def login(request, user, backend=None):
    """
    在请求中保留用户标识和后端。这样一来，用户就不必在每个请求上重新认证。 请注意，匿名会话期间的数据集在用户登录时会保留。
    """
    session_auth_hash = ''
    if user is None:
        user = request.user
    if hasattr(user, 'get_session_auth_hash'):
        session_auth_hash = user.get_session_auth_hash()

    if SESSION_KEY in request.session:
        # user.pk即user在数据库中的id
        if _get_user_session_key(request) != user.pk or (
                session_auth_hash and
                not constant_time_compare(request.session.get(HASH_SESSION_KEY, ''), session_auth_hash)):
            # To avoid reusing another user's session, create a new, empty
            # session if the existing session corresponds to a different
            # authenticated user.
            request.session.flush()
    else:
        request.session.cycle_key()

    try:
        backend = backend or user.backend
    except AttributeError:
        backends = _get_backends(return_tuples=True)
        if len(backends) == 1:
            _, backend = backends[0]
        else:
            raise ValueError(
                'You have multiple authentication backends configured and '
                'therefore must provide the `backend` argument or set the '
                '`backend` attribute on the user.'
            )

    request.session[SESSION_KEY] = user._meta.pk.value_to_string(user)
    request.session[BACKEND_SESSION_KEY] = backend
    request.session[HASH_SESSION_KEY] = session_auth_hash
    if hasattr(request, 'user'):
        request.user = user
    rotate_token(request)
    user_logged_in.send(sender=user.__class__, request=request, user=user)
```
登录时，需要获取到用户的session_auth_hash(session_auth_hash，在django中为HMAC，是一种与密钥相关的哈希运算消息认证码，HMAC运算利用哈希算法，以一个密钥和一个消息为输入，生成一个消息摘要作为输出)，user.get_session_auth_hash()函数：
```python
    def get_session_auth_hash(self):
        """
        Return an HMAC of the password field.
        """
        key_salt = "django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash"
        return salted_hmac(key_salt, self.password).hexdigest()
```
session_auth_hash的作用是判断用户请求时携带的session_key是否在服务器的session中。如果传递来的session_key经过转换之后与user.pk不一致，或者与服务器中对应的session_key不一致，则将该session清除掉。
```python
    if SESSION_KEY in request.session:
        if _get_user_session_key(request) != user.pk or (
                session_auth_hash and
                not constant_time_compare(request.session.get(HASH_SESSION_KEY, ''), session_auth_hash)):
            # To avoid reusing another user's session, create a new, empty
            # session if the existing session corresponds to a different
            # authenticated user.
            request.session.flush()
        else:
            request.session.cycle_key()
```
cycle_key()函数：
```python
    def cycle_key(self):
        """
        Creates a new session key, while retaining the current session data.
        """
        data = self._session
        key = self.session_key
        self.create()  # 将session保存在数据库中
        self._session_cache = data
        if key:
            self.delete(key)
```
session保存之后，在request.session中设置SESSION_KEY, BACKEND_SESSION_KEY, HASH_SESSION_KEY，通过rotate_token()函数重置CSRF。
```python
def rotate_token(request):
    """
    Changes the CSRF token in use for a request - should be done on login
    for security purposes.
    """
    request.META.update({
        "CSRF_COOKIE_USED": True,
        "CSRF_COOKIE": _get_new_csrf_token(),
    })
    request.csrf_cookie_needs_reset = True
```
最后是信号的传递，这个会专门做一节。
回过头来，我们看到request会有一个session项，这个是哪里来的呢？
这就是django的中间件技术，Django的中间件框架，是django处理请求和响应的一套钩子函数的集合。传统的django视图模式一般是这样的：http请求->view->http响应，而加入中间件框架后，则变为：http请求->中间件处理->app->中间件处理->http响应。而在django中这两个处理分别对应process_request和process_response函数，这两个钩子函数将会在特定的时候被触发。
那么session的中间件是什么呢？
我们创建django项目时，settings.py中会有默认的session中间件，SessionMiddleware：
```python
class SessionMiddleware(object):
    def __init__(self):
        engine = import_module(settings.SESSION_ENGINE)
        self.SessionStore = engine.SessionStore

    def process_request(self, request):
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        request.session = self.SessionStore(session_key)

    def process_response(self, request, response):
        """
        If request.session was modified, or if the configuration is to save the
        session every time, save the changes and set a session cookie or delete
        the session cookie if the session has been emptied.
        """
        try:
            accessed = request.session.accessed
            modified = request.session.modified
            empty = request.session.is_empty()
        except AttributeError:
            pass
        else:
            # First check if we need to delete this cookie.
            # The session should be deleted only if the session is entirely empty
            if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
                response.delete_cookie(settings.SESSION_COOKIE_NAME,
                    domain=settings.SESSION_COOKIE_DOMAIN)
            else:
                if accessed:
                    patch_vary_headers(response, ('Cookie',))
                if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
                    if request.session.get_expire_at_browser_close():
                        max_age = None
                        expires = None
                    else:
                        max_age = request.session.get_expiry_age()
                        expires_time = time.time() + max_age
                        expires = cookie_date(expires_time)
                    # Save the session data and refresh the client cookie.
                    # Skip session save for 500 responses, refs #3881.
                    if response.status_code != 500:
                        try:
                            request.session.save()
                        except UpdateError:
                            # The user is now logged out; redirecting to same
                            # page will result in a redirect to the login page
                            # if required.
                            return redirect(request.path)
                        response.set_cookie(settings.SESSION_COOKIE_NAME,
                                request.session.session_key, max_age=max_age,
                                expires=expires, domain=settings.SESSION_COOKIE_DOMAIN,
                                path=settings.SESSION_COOKIE_PATH,
                                secure=settings.SESSION_COOKIE_SECURE or None,
                                httponly=settings.SESSION_COOKIE_HTTPONLY or None)
        return response
```
在请求过来之后，django中间件会发挥作用，process_request会在请求的Cookie中取出session_key，并把一个新的session对象赋给request.session，而在返回响应时，process_response则判断session是否被修改或者是否过期，来更新session信息。