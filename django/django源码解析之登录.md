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