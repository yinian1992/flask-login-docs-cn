===========
Flask-Login
===========
.. currentmodule:: flask.ext.login

Flask-Login 为 Flask 提供了会话管理。它处理日常的登入、登出并长期保留用户会话。

它会：

- 存储会话中活动用户的 ID，并允许你随意登入登出。
- 让你限制已登入（或已登出）用户访问视图。
- 实现棘手的“记住我”功能。
- 保护用户会话免遭 Cookie 盗用。
- 随后可能会与 Flask-Principal 或其它认证扩展集成。

无论如何，它不会：

- 限制你使用特定的数据库或其它存储方法。如何加载用户完全由你决定。
- 限制用户名和密码、OpenID 或其它认证方法的使用。
- 处理“登入或未登入”之外的权限。
- 处理用户注册信息或账号恢复。

.. contents::
   :local:
   :backlinks: none


配置你的应用
============================
对一个使用 Flask-Login 的应用，最重要的部分就是 `LoginManager` 类。你应该在代
码中的某处为你的应用创建一个它的实例，像这样::

    login_manager = LoginManager()

登录管理器包含了让你的应用与 Flask-Login 协同工作的代码，诸如如何从 ID 加载一个
用户、当用户需要登录时跳转到哪里等等。

一旦创建了实际的应用对象，你可以这样配置它来实现登录功能::

    login_manager.setup_app(app)


它如何工作
============
你需要提供一个 `~LoginManager.user_loader` 回调。这个回调用于从会话中存储的用户
ID 重新加载用户对象。它应该接受一个用户的 `unicode` ID，并返回相应的用户对象。
例如::

    @login_manager.user_loader
    def load_user(userid):
        return User.get(userid)

如果 ID 无效，它应该返回 `None` （ **而不是抛出异常** ）。（在这种情况下，ID 会
被手动从会话中移除且处理会继续。）

用户通过验证后，用 `login_user` 函数来登入他们。例如::

    @app.route("/login", methods=["GET", "POST"])
    def login():
        form = LoginForm()
        if form.validate_on_submit():
            # login and validate the user...
            login_user(user)
            flash("Logged in successfully.")
            return redirect(request.args.get("next") or url_for("index"))
        return render_template("login.html", form=form)

就这么简单。你可以之后通过 `current_user` 代理访问已登入的用户。需要用户登入
的视图可以用 `login_required` 装饰器来装饰::

    @app.route("/settings")
    @login_required
    def settings():
        pass

当用户要登出时::

    @app.route("/logout")
    @login_required
    def logout():
        logout_user()
        return redirect(somewhere)

他们会被登出，且他们会话产生的任何 cookie 都会被清理干净。


你的用户类
===============
用于表示用户的类需要实现这些方法：

`is_authenticated()`
    当用户通过验证时，也即提供有效证明时返回 `True` 。（只有通过验证的用户
    会满足 `login_required` 的条件。）

`is_active()`
    如果这是一个活动用户且通过验证，账户也已激活，未被停用，也不符合任何你
    的应用拒绝一个账号的条件，返回 `True` 。不活动的账号可能不会登入（当然，
    是在没被强制的情况下）。

`is_anonymous()`
    如果是一个匿名用户，返回 `True` 。（真实用户应返回 `False` 。）

`get_id()`
    返回一个能唯一识别用户的，并能用于从 `~LoginManager.user_loader` 回调中
    加载用户的 `unicode` 。注意着 **必须** 是一个 `unicode` ——如果 ID 原本是
    一个 `int` 或其它类型，你需要把它转换为 `unicode` 。

要简便地实现用户类，你可以从 `UserMixin` 继承，它提供了对所有这些方法的默认
实现。（虽然这不是必须的。）

自定义登入过程
=============================
默认，在用户未登入的情况下试图访问一个 `login_required` 视图，Flask-Login 会
闪现一条消息并把他们重定向到登入视图。（如果未设定登入视图，会以 401 错误退
出。）

登入视图的名称可以设置为 `LoginManager.login_view` 。例如::

    login_manager.login_view = "users.login"

默认闪现的消息是 ``Please log in to access this page.`` 要自定义消息，设置
`LoginManager.login_message`::

    login_manager.login_message = u"Bonvolu ensaluti por uzi tio paĝo."

要自定义消息的分类，设置 `LoginManager.login_message_category`::

    login_manager.login_message_category = "info"

当重定向到登入视图，它的请求字符串中会有一个 ``next`` 变量，值为用户之前试图
访问的页面。

如果你想要进一步自定义登入过程，用 `LoginManager.unauthorized_handler` 装饰
一个函数::

    @login_manager.unauthorized_handler
    def unauthorized():
        # do stuff
        return a_response


匿名用户 
===============
默认，当用户还没有登入时， `current_user` 被设置为一个 `AnonymousUser` 对象。
它包含下列属性：

- `is_active` 和 `is_authenticated` 返回 `False`
- `is_active` 返回 `True`
- `get_id` 返回 `None`

如果你有自定义匿名用户的需求（例如需要一个权限域），你可以向 `LoginManager`
提供一个创建匿名用户的回调（类或工厂函数）::

    login_manager.anonymous_user = MyAnonymousUser


记住我
===========
“记住我”功能的实现很棘手。尽管如此，Flask-Login 几乎透明地实现了这——只需要向
`login_user` 调用传递 ``remember=True`` 。一个 cookie 就会存储在用户的电脑上，
且之后如果会话中没有用户 ID，Flask-Login 会自动从那个 cookie 上恢复用户 ID。
这个 cookie 是防篡改的，所以如果用户篡改了它（插入其它用户的 ID 来替代自己
的），这个 cookie 只不过会被拒绝，就如同没有一样。

该层功能是被自动实现的。但你能（且应该，如果你的应用处理任何敏感的数据）提供
额外基础工作来增强你记住的 cookie 的安全性。

可选令牌
------------------
使用用户 ID 作为记住的令牌值不一定是安全的。更安全的方法是使用用户名和密
码联合的 hash 值，或类似的东西。要添加一个额外的令牌，向你的用户对象添加
一个方法：

`get_auth_token()`
    返回用户的认证令牌（返回为 `unicode` ）。这个认证令牌应能唯一识别用户，且
    不易通过用户的公开信息，如 UID 和名称来猜测出——同样也不应暴露这些信息。

相应地，你应该在 `LoginManager` 上设置一个 `~LoginManager.token_loader` 函数，
它接受令牌（存储在 cookie 中）作为参数并返回合适的 `User` 对象。

`make_secure_token` 函数用于便利创建认证令牌。它会连接所有的参数，然后用应用的
密钥来 HMAC 它确保最大的密码学安全。（如果你永久地在数据库中存储用户令牌，那么
你会希望向令牌中添加随机数据来阻碍猜测。）

如果你的应用使用密码来验证用户，在认证令牌中包含密码（或你应使用的加盐值的密码
hash ）能确保若用户更改密码，他们的旧认证令牌会失效。

活跃登录
------------
当用户登入，他们的会话被标记为“活跃”，这表明他们确实在那个会话中通过验证。当
他们的会话被销毁且他们使用“记住我” cookie 重新登入，它会被标记为“非活跃”。
`login_required` 并不区分这两种状态，这对于大多数页面已经足够了。但像更改某人
的个人信息这样的敏感操作应需要一个活跃的登入。（像修改密码这样的操作总是需要
密码，无论是否重登入。）

`fresh_login_required` ，除验证用户登入外，也会确保用户的登入是活跃的。如果不
是，它会把用户送到可以重输入验证条件的页面。你可以用自定义 `login_required`
相同的方法自定义这个行为，设定 `LoginManager.refresh_view` 、 
`~LoginManager.needs_refresh_message` 和
`~LoginManager.needs_refresh_message_category`::

    login_manager.refresh_view = "accounts.reauthenticate"
    login_manager.needs_refresh_message = (
        u"To protect your account, please reauthenticate to access this page."
    )
    login_manager.needs_refresh_message_category = "info"

或者提供你自己的回调来处理活跃刷新::

    @login_manager.needs_refresh_handler
    def refresh():
        # do stuff
        return a_response

要重新把会话标记为活跃的，调用 `confirm_login` 函数。


Cookie 设置
---------------
Cookie 的细节可以在应用设置中定义。

=========================== =================================================
`REMEMBER_COOKIE_NAME`      存储“记住我”信息的 cookie 名。 **默认值：**
                            ``remember_token``
`REMEMBER_COOKIE_DURATION`  Cookie 过期时间，为一个 `datetime.timedelta` 对
                            象。 **默认值：** 365 天（一非闰阳历年）
`REMEMBER_COOKIE_DOMAIN`    如果“记住我” cookie 应跨域，在此处设置域名值（即
                            ``.example.com`` 会允许 ``example`` 下所有子域
                            名）。 **默认值：** `None`
=========================== =================================================


会话保护
==================
当上述特性保护“记住我”令牌免遭 cookie 窃取时，会话 cookie 仍然是脆弱的。
Flask-Login 包含了会话保护来帮助阻止用户会话被盗用。

你可以在 `LoginManager` 上和应用配置中配置会话保护。如果它被启用，它可以在
`basic` 或 `strong` 两种模式中运行。要在 `LoginManager` 上设置它，设置
`~LoginManager.session_protection` 属性为 ``"basic"`` 或 ``"strong"``::

    login_manager.session_protection = "strong"

或禁用它:

    login_manager.session_protection = None

默认，它被激活为 ``"basic"`` 模式。它可以在应用配置中设定
`SESSION_PROTECTION` 为 `None` 、 ``"basic"`` 或 ``"strong"`` 来禁用。

当启用了会话保护，每个请求，它生成一个用户电脑的标识（基本上是 IP 地址和
User Agent 的 MD5 hash 值）。如果会话不包含相关的标识，则存储生成的。如果
存在标识，则匹配生成的，之后请求可用。

在 `basic` 模式下或会话是永久的，如果该标识未匹配，会话会简单地被标记为非活
跃的，且任何需要活跃登入的东西会强制用户重新验证。（当然，你必须已经使用了活
跃登入机制才能奏效。）

在 `strong` 模式下的非永久会话，如果该标识未匹配，整个会话（记住的令牌如果存
在，则同样）被删除。

API 文档
=================
该文档由 Flask-Login 的源码自动生成。


Configuring Login
-----------------
.. autoclass:: LoginManager
   
   .. automethod:: setup_app
   
   .. automethod:: unauthorized
   
   .. automethod:: needs_refresh
   
   .. rubric:: 常规配置
   
   .. automethod:: user_loader
   
   .. automethod:: token_loader
   
   .. attribute:: anonymous_user
   
      一个生成匿名用户的类或工厂函数，匿名用户在无人登入的时候使用。
   
   .. rubric:: `unauthorized` 配置
   
   .. attribute:: login_view
   
      当用户需要登入的时候跳转到的视图的名字。（如果你的认证机构在应用之外，
      也可以是一个绝对 URL。）

   .. attribute:: login_message
   
      当用户重定向到登入页面时要闪现的消息。
   
   .. automethod:: unauthorized_handler
   
   .. rubric:: `needs_refresh` 配置
   
   .. attribute:: refresh_view

      当用户需要重新验证时跳转到的视图的名称。
   
   .. attribute:: needs_refresh_message
      
      当用户重定向到重验证页面时闪现的消息。

   .. automethod:: needs_refresh_handler


登入机制
----------------
.. data:: current_user

   当前用户的代理。

.. autofunction:: login_fresh

.. autofunction:: login_user

.. autofunction:: logout_user

.. autofunction:: confirm_login


视图保护
----------------
.. autofunction:: login_required

.. autofunction:: fresh_login_required


用户对象辅助
-------------------
.. autoclass:: UserMixin
   :members:

.. autoclass:: AnonymousUser
   :members:


实用工具
---------
.. autofunction:: login_url

.. autofunction:: make_secure_token


信号
-------
欲知在代码中如何使用这些信号，请查阅 `Flask 文档的信号部分`_ 。

.. data:: user_logged_in

   当一个用户登入的时候发出。除应用（信号的发送者）之外，它还传递正登
   入的用户 `user` 。

.. data:: user_logged_out

   当一个用户登出的时候发出。除应用（信号的发送者）之外，它还传递正登
   出的用户 `user` 。

.. data:: user_login_confirmed

   当用户的登入被证实，把它标记为活跃的。（它不用于常规登入的调用。）
   它不接受应用以外的任何其它参数。

.. data:: user_unauthorized

   当 `LoginManager` 上的 `unauthorized` 方法被调用时发出。它不接受应用
   以外的任何其它参数。

.. data:: user_needs_refresh

   当 `LoginManager` 上的 `needs_refresh` 方法被调用时发出。它不接受应用
   以外的任何其它参数。

.. data:: session_protected

   当会话保护起作用时，且会话被标记为非活跃或删除时发出。它不接受应用以外
   的任何其它参数。

.. _Flask 文档的信号部分: http://flask.pocoo.org/docs/signals/
