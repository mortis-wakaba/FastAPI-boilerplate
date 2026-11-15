<a id="migrating-passwords"></a>

# 迁移密码

迁入已有用户时也要迁入密码哈希。哈希无法直接从一种方案转换成另一种，只有原始密码可以重新哈希，而迁移时通常没有原始密码。`legacy_verifiers` 解决这一问题：旧哈希原样导入，用户首次登录成功后替换为 crudauth 哈希，无需重置密码，也不会同时存储两套哈希。

<a id="why-the-old-hashes-dont-just-work"></a>

## 为什么旧哈希不能直接使用

crudauth 存储 `bcrypt(base64(sha256(password)))`。SHA-256 是固定宽度转换，不是工作因子；bcrypt 忽略 72 字节后的输入，先哈希可以让全部密码参与比较。因此其他库通过 `bcrypt.hashpw(password, salt)` 生成的哈希，虽然也是 bcrypt，却**不会**通过 crudauth 自身校验，直接导入会导致登录 401。

<a id="wiring-a-verifier"></a>

## 接入校验器

校验器签名为 `(plain_password, hashed_password) -> bool`，crudauth 为普通 bcrypt 提供内置实现：

```python
# infrastructure/auth/setup.py
from crudauth.utils import verify_plain_bcrypt

auth = CRUDAuth(
    ...,
    legacy_verifiers=[verify_plain_bcrypt],
)
```

登录先运行自身校验，失败后才在线程池中逐个调用旧校验器。匹配后完成登录，并将 `hashed_password` 改为 crudauth 格式，因此每个账户只需成功使用旧校验器一次。

| 旧方案 | 传入内容 |
|--------|----------|
| 普通 bcrypt，如 `bcrypt.hashpw`、passlib 的 `bcrypt` | `verify_plain_bcrypt` |
| passlib `bcrypt_sha256`、Django `BCryptSHA256PasswordHasher` | 自定义校验器，因为预哈希方式不同 |
| Django 默认 PBKDF2、Argon2、scrypt、旧 MD5 / SHA-1 | 自定义函数调用对应库 |

将普通函数加入列表：

```python
from passlib.hash import django_pbkdf2_sha256


def verify_django_pbkdf2(plain_password: str, hashed_password: str) -> bool:
    """Check a password against a hash Django's default hasher wrote."""
    try:
        return django_pbkdf2_sha256.verify(plain_password, hashed_password)
    except ValueError:
        return False


auth = CRUDAuth(..., legacy_verifiers=[verify_django_pbkdf2, verify_plain_bcrypt])
```

可列出多个校验器，适合多来源迁移。校验器异常视为不匹配并记录日志，避免把损坏实现误认为所有用户都输错密码。部署后关注 `legacy verifier ... raised`。

!!! note "自定义校验器可能泄露用户名耗时差异"
    用户名未知时，crudauth 以 bcrypt 虚拟哈希运行全部校验器。其他哈希方案的校验器可能立即返回，而真实迁移账户需要完整校验耗时。配置期间的时间差可能泄露用户名存在性，因此迁移完成应移除。`verify_plain_bcrypt` 不受影响，因为虚拟哈希同为 bcrypt，耗时相近。

!!! warning "校验器只能接受真实密码"
    crudauth 自身哈希也是 bcrypt，朴素的普通 bcrypt 校验会接受其内部使用的 base64 SHA-256 摘要。攻击者若从其他来源得到无盐 SHA-256 摘要，可能把摘要当作密码登录。`verify_plain_bcrypt` 因此拒绝形如该摘要的输入：44 个 base64 字符、解码为 32 字节。自定义校验器应保留相同保护，或使用格式不可能与 bcrypt 混淆的库。

<a id="importing-the-rows"></a>

## 导入记录

将旧哈希直接复制到 `user.hashed_password`，无需标记 legacy。crudauth 自身验证失败后会回退到旧校验器。

```python
await crud_users.create(
    db=db,
    object=UserCreateInternal(
        name=row.full_name,
        username=row.username,
        email=row.email,
        hashed_password=row.password,  # the old system's hash, as it is
    ),
)
```

需要注意：

- **密码策略不适用于哈希**。`PASSWORD_MIN_LENGTH` 等仅在注册、修改、重置和 `/set-password` 设置密码时检查。因此不符合当前策略的旧密码仍可登录，并按原值重新哈希。若必须执行新策略，应要求重置，而非依赖校验器。
- **没有密码的账户**，如一直用 OAuth 登录的账户，应存 `make_unusable_password()`，不能存空字符串。`/api/v1/auth/login` 会拒绝它，用户通过提供方登录，或使用先验证邮箱的重置链接设置密码，见[密码](sessions.md#passwords)。

<a id="finishing-the-migration"></a>

## 完成迁移

每次登录转换一个账户，所有活跃账户都登录后即可移除校验器。crudauth 的哈希是对 44 字符 base64 摘要的 bcrypt，外观无法与普通 bcrypt 区分。因此导入时应在自定义列或附表记录迁移状态，并在登录成功后清除标记，可使用 crudauth 的 `on_after_login` 钩子，与[拒绝审计](sessions.md#what-a-refusal-records)放在一起。

剩余的是未回访账户，可以停用或要求重置。`POST /api/v1/auth/password/reset-request` 发送邮件，重置会写入 crudauth 格式。

校验器只保留到完成迁移为止。它是额外验证路径，每次自身校验失败都会在线程中产生额外开销。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 传入 `legacy_verifiers` 的 `auth = CRUDAuth(...)` | `backend/src/infrastructure/auth/setup.py` |
| `verify_plain_bcrypt`、`make_unusable_password` | crudauth 的 `crudauth.utils` |
| 旧密码登录和重新哈希测试 | `backend/tests/integration/auth/test_legacy_passwords.py` |

---

[← 会话](sessions.md){ .md-button } [用户管理 →](user-management.md){ .md-button .md-button--primary }
