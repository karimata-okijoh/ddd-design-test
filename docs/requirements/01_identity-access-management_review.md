# ID管理・ログイン管理（IAM）設計 レビュー

**レビュー日**: 2026-03-06
**対象ドキュメント**: `01_identity-access-management.md`
**ステータス**: 第2回レビュー完了（2026-03-06）・要対応8件

| 懸念点 | 対応状況 | 備考 |
|--------|----------|------|
| 1. User集約のIdentity依存 | ⏭️ 対応見送り | 仕様書の意図的なトレードオフ。ASP.NET Core Identityとの統合を優先した設計方針につき変更しない |
| 2. RefreshToken集約設計 | ⏭️ 対応見送り | Session集約として分離する設計は合理的（ユーザーロード時のN+1回避、スケーラビリティ）。現設計を維持 |
| 3. テナント登録時の整合性 | ✅ 反映済み | 単一トランザクション（C案）を初期実装方針として仕様に追加 |
| 4. ログインフローのセキュリティ | ✅ 反映済み | テナント整合性チェックをパスワード検証後に移動。レート制限注記を追加 |
| 5. AuditLogの集約設計 | ⏭️ 対応見送り | 仕様書は既にApplication層からIAuditLogRepository経由で呼ぶ設計であり問題なし |
| 6. 未決定事項の優先度 | ✅ 反映済み | 未決定事項は仕様書に既存。追加ユースケース・セキュリティ・運用・パフォーマンス・インデックスを反映 |

---

## 第2回レビュー（2026-03-06）

反映済み仕様書に対する再検証結果。

| # | 優先度 | 項目 | 対応状況 |
|---|--------|------|----------|
| R2-1 | 🔴 高 | パスワードリセットトークンのドメインモデルが未定義 | 未対応 |
| R2-2 | 🔴 高 | ログインフロー: null user を CheckPasswordSignInAsync に渡せない | 未対応 |
| R2-3 | 🟡 中 | システム管理者ロールが未定義（T-03/T-04 の実行者） | 未対応 |
| R2-4 | 🟡 中 | レイヤー構成に新規ユースケースファイルが未反映 | 未対応 |
| R2-5 | 🟡 中 | ロックアウト閾値が「未確定」とコード確定値で矛盾 | 未対応 |
| R2-6 | 🟢 低 | RefreshToken集約 / Session集約の名称不一致 | 未対応 |
| R2-7 | 🟢 低 | P-01 Eager Loading の記述が不正確 | 未対応 |
| R2-8 | 🟢 低 | 新規ユースケースの監査ログ記録タイミングが未記載 | 未対応 |

### R2-1. パスワードリセットトークンのドメインモデルが未定義 🔴

A-04/A-05 を実装するには一時トークン（有効期限・使用済み管理）が必要だが、どこにも定義がない。

**決定が必要な事項**:
```
選択肢A: ASP.NET Core Identity の GeneratePasswordResetTokenAsync に委譲（推奨）
  → UserManager がトークン生成・検証を管理
  → 追加モデル不要だが、メール送信インフラが必要
  → Infrastructure/Auth/ に PasswordResetEmailService を追加

選択肢B: 独自の一時トークン管理テーブルを設ける
  → Domain/Auth/PasswordResetToken.cs を追加
  → 有効期限・使用済みフラグを管理
```

**推奨**: 選択肢A（Identity に委譲）。追加モデルなしで実装できる。

### R2-2. ログインフロー: null user のダミーチェック問題 🔴

`CheckPasswordSignInAsync(null, ...)` は `NullReferenceException` を投げる。
現状の仕様記述では実装不可能。

**推奨する修正フロー**:
```
LoginUseCase
  │
  ├─ UserManager.FindByEmailAsync(email)
  │
  ├─ user が null の場合
  │     → _passwordHasher.VerifyHashedPassword("", password)  // タイミング均一化
  │     → 認証失敗（同一エラー）
  │
  ├─ user が存在する場合
  │     → SignInManager.CheckPasswordSignInAsync(user, password, lockoutOnFailure: true)
  │     → 失敗の場合 → 認証失敗（同一エラー）
  │
  ├─ テナント整合性チェック ...
```

### R2-3. システム管理者ロールが未定義 🟡

T-03（テナント停止）・T-04（テナント削除）の実行者「システム管理者」に対応するロールが未定義。
ロール定義は `TenantAdmin` / `Member` のみ。

**決定が必要な事項**:
- `SystemAdmin` ロールを追加するか
- テナント操作は API キー認証など別方式にするか

### R2-4. レイヤー構成に新規ユースケースが未反映 🟡

追加された 10 件のユースケースのファイルが Application/ 層に記載されていない。

```
Application/
  ├── Auth/
  │   ├── PasswordResetRequestUseCase.cs   # A-04
  │   ├── PasswordResetExecuteUseCase.cs   # A-05
  │   ├── VerifyEmailUseCase.cs            # A-06
  │   └── LogoutAllDevicesUseCase.cs       # A-07
  ├── Tenant/
  │   ├── UpdateTenantUseCase.cs           # T-02
  │   ├── SuspendTenantUseCase.cs          # T-03
  │   └── DeleteTenantUseCase.cs           # T-04
  └── User/
      ├── ListUsersUseCase.cs              # U-04
      ├── DeleteUserUseCase.cs             # U-05
      ├── ChangeUserRoleUseCase.cs         # U-06
      └── UnlockUserUseCase.cs            # U-07
```

### R2-5. ロックアウト閾値の矛盾 🟡

未決定事項テーブルで「**未確定**」とあるが、セキュリティ考慮事項のコードでは 5回・30分 と具体的な値が記載済み。

**推奨**: 未決定事項テーブルの当該行を「**確定**（5回失敗で30分ロック）」に変更する。

### R2-6. RefreshToken集約 / Session集約の名称不一致 🟢

- 境界コンテキスト図: `SA["Session 集約"]`
- ドメインモデルの見出し: `### RefreshToken 集約`
- レイヤー構成: `Domain/Session/`

**推奨**: 「Session 集約」に統一（より意味が広く、将来的な拡張に対応しやすい）。

### R2-7. P-01 Eager Loading の記述が不正確 🟢

`.Include(u => u.Roles)` は ASP.NET Core Identity の標準 many-to-many 構造では動作しない。

```csharp
// 正しくは（ApplicationUser に UserRoles ナビゲーションプロパティを追加した上で）
.Include(u => u.UserRoles)
    .ThenInclude(ur => ur.Role)
```

### R2-8. 新規ユースケースの監査ログ記録が未記載 🟢

以下のイベントが記録対象一覧・記録タイミング例に未追加:

| ユースケース | 追加すべきイベント種別 |
|------------|----------------------|
| A-04 パスワードリセット要求 | `auth.password.reset.request` |
| A-05 パスワードリセット実行 | `auth.password.reset.complete` |
| A-07 全デバイスログアウト | `auth.logout.all_devices` |
| U-05 ユーザー削除 | `user.deleted` |
| U-06 ロール変更 | `user.role.changed`（既存） |
| T-03 テナント停止 | `tenant.suspended`（既存） |
| T-04 テナント削除 | `tenant.deleted` |

---

## 全体評価

DDDアーキテクチャとASP.NET Core Identityの責務分離が明確で、よく整理された設計です。以下、改善提案と懸念点を指摘します。

---

## 強み

- ASP.NET Core Identityとドメイン層の責務分離が明確
- 監査ログの抽象化設計により将来の拡張性を確保
- RefreshTokenをドメインモデルとして適切に管理
- テナント境界の整合性チェックを考慮

---

## 重要な懸念点

### 1. User集約の設計上の問題 ⚠️ 高優先度 ⏭️ 対応見送り

**問題**: `ApplicationUser`がIdentityUserを継承しているため、ドメイン層がインフラ層（ASP.NET Core Identity）に依存している

```
Domain/User/ApplicationUser.cs : IdentityUser<Guid>  ← インフラへの依存
```

**DDD原則違反**: ドメイン層は最も内側の層であり、外部フレームワークに依存すべきではない

**推奨アプローチ**:
- ドメイン層に純粋な`User`エンティティを定義
- Infrastructure層に`ApplicationUser : IdentityUser<Guid>`を配置
- リポジトリ層でドメインモデル⇔Identityモデルのマッピングを実施

```
Domain/User/User.cs                              # 純粋なドメインモデル
Infrastructure/Identity/ApplicationUser.cs       # IdentityUser継承
Infrastructure/Identity/UserRepository.cs        # マッピング処理
```

**実装例**:

```csharp
// Domain層
namespace Domain.User
{
    public class User
    {
        public UserId Id { get; private set; }
        public TenantId TenantId { get; private set; }
        public Email Email { get; private set; }
        public DisplayName DisplayName { get; private set; }
        public DateTime CreatedAt { get; private set; }
        
        // ドメインロジック
        public void ChangeDisplayName(DisplayName newName) { ... }
    }
}

// Infrastructure層
namespace Infrastructure.Identity
{
    public class ApplicationUser : IdentityUser<Guid>
    {
        public Guid TenantId { get; set; }
        public string DisplayName { get; set; }
        public DateTime CreatedAt { get; set; }
        
        // Domainモデルへの変換
        public User ToDomain() { ... }
        public static ApplicationUser FromDomain(User user) { ... }
    }
}
```

---

### 2. RefreshTokenの集約設計 ⚠️ 中優先度 ⏭️ 対応見送り

**懸念**: RefreshTokenを独立した集約として定義しているが、実際にはUserの一部として扱うべきでは？

**理由**:
- RefreshTokenはUserなしでは存在意義がない
- トランザクション境界を考えると、User集約の一部として管理する方が整合性を保ちやすい
- 1ユーザーが複数のRefreshTokenを持つ関係性

**現在の設計**:
```
RefreshToken集約（独立）
  ├─ TokenId
  ├─ UserId（参照）
  └─ ...
```

**推奨設計**:
```csharp
class User {
    private List<RefreshToken> _refreshTokens;
    
    public RefreshToken IssueRefreshToken(DateTime expiresAt) 
    {
        var token = RefreshToken.Create(this.Id, this.TenantId, expiresAt);
        _refreshTokens.Add(token);
        return token;
    }
    
    public void RevokeRefreshToken(TokenId tokenId) 
    {
        var token = _refreshTokens.FirstOrDefault(t => t.TokenId == tokenId);
        token?.Revoke();
    }
    
    public void RevokeAllRefreshTokens() 
    {
        foreach (var token in _refreshTokens)
            token.Revoke();
    }
}
```

**メリット**:
- トランザクション整合性の保証
- 集約ルートを通じた操作の強制
- ビジネスルール（例：同時発行上限）の実装が容易

---

### 3. テナント登録時の整合性 ⚠️ 高優先度 ✅ 反映済み

**問題**: T-01「テナント登録（初回管理者ユーザーも同時作成）」で2つの集約を同時作成

**懸念**:
- Tenant集約とUser集約を同一トランザクションで作成する必要がある
- DDDでは異なる集約間のトランザクション整合性は結果整合性が推奨される
- トランザクション失敗時のロールバック戦略が不明確

**推奨アプローチ（オプション）**:

**A案: ドメインイベント + 結果整合性**
```
1. Tenantを作成 → TenantCreated イベント発行
2. イベントハンドラで初回管理者ユーザーを作成
3. 失敗時は補償トランザクションでTenantを削除
```

**B案: Sagaパターン**
```
RegisterTenantSaga
  ├─ Step 1: Tenant作成
  ├─ Step 2: 初回管理者User作成
  └─ 失敗時: 補償トランザクション実行
```

**C案: 単一トランザクション（実用的）**
```csharp
// Application層
public async Task<Result> RegisterTenantAsync(RegisterTenantRequest request)
{
    using var transaction = await _dbContext.BeginTransactionAsync();
    try
    {
        // 1. Tenant作成
        var tenant = Tenant.Create(request.TenantName);
        await _tenantRepository.SaveAsync(tenant);
        
        // 2. 初回管理者User作成
        var adminUser = await _userManager.CreateAsync(
            new ApplicationUser { TenantId = tenant.Id, ... },
            request.AdminPassword
        );
        await _userManager.AddToRoleAsync(adminUser, "TenantAdmin");
        
        await transaction.CommitAsync();
        return Result.Success();
    }
    catch
    {
        await transaction.RollbackAsync();
        return Result.Failure("テナント登録に失敗しました");
    }
}
```

**推奨**: 初期実装はC案（単一トランザクション）、将来的にA案への移行を検討

---

### 4. ログインフローのセキュリティ ⚠️ 高優先度 ✅ 反映済み

**問題**: テナント整合性チェックのタイミングによる情報漏洩リスク

**現在のフロー**:
```
├─ UserManager.FindByEmailAsync(email)
│     → null の場合 → 認証失敗
├─ テナント整合性チェック（user.TenantId == requestTenantId）
│     → 不一致の場合 → 認証失敗
├─ SignInManager.CheckPasswordSignInAsync(...)
```

**セキュリティリスク**: 
- パスワードチェック前にテナント不一致エラーを返すと、「このメールアドレスは別テナントに存在する」という情報が漏洩
- タイミング攻撃によるユーザー列挙が可能

**推奨フロー**:
```
LoginUseCase
  │
  ├─ UserManager.FindByEmailAsync(email)
  │     → null の場合でも処理を継続（ダミーチェック）
  │
  ├─ SignInManager.CheckPasswordSignInAsync(user, password, lockoutOnFailure: true)
  │     → パスワード検証を先に実行
  │
  ├─ テナント整合性チェック（user.TenantId == requestTenantId）
  │     → 不一致の場合も「認証失敗」として同一エラーを返す
  │
  ├─ すべてのチェックが成功した場合のみ
  │     → UserManager.GetRolesAsync(user)
  │     → JwtService.Issue(...)
  │     → RefreshToken.Create(...)
  │
  └─ すべての失敗ケースで同一エラーメッセージ
        → "メールアドレスまたはパスワードが正しくありません"
```

**追加推奨**:
- レート制限（IP単位、メールアドレス単位）
- 失敗時の遅延応答（タイミング攻撃対策）

---

### 5. AuditLogの集約設計 ⚠️ 中優先度 ⏭️ 対応見送り

**問題**: AuditLogを独立したエンティティとして定義しているが、どの集約に属するか不明確

**現在の設計**:
```
AuditLog（append-only）
  ├─ Id
  ├─ TenantId
  ├─ UserId
  └─ ...
```

**懸念**:
- AuditLogは集約ルートではなく、ドメインイベントの記録
- 直接AuditLogを作成するのではなく、ドメインイベントから生成すべき

**推奨アプローチ**:

```
ドメイン層
  └─ ドメインイベント発行
        ├─ UserLoggedIn
        ├─ LoginFailed
        ├─ TenantCreated
        └─ ...

アプリケーション層
  └─ ドメインイベントハンドラ
        └─ AuditLogEventHandler
              └─ IAuditLogRepository.AppendAsync()
```

**実装例**:
```csharp
// Domain層
public class UserLoggedIn : DomainEvent
{
    public UserId UserId { get; }
    public TenantId TenantId { get; }
    public string IpAddress { get; }
    public DateTime OccurredAt { get; }
}

// Application層
public class AuditLogEventHandler : 
    INotificationHandler<UserLoggedIn>,
    INotificationHandler<LoginFailed>
{
    public async Task Handle(UserLoggedIn @event)
    {
        await _auditLogRepository.AppendAsync(new AuditLog
        {
            EventType = "auth.login.success",
            UserId = @event.UserId,
            TenantId = @event.TenantId,
            IpAddress = @event.IpAddress,
            OccurredAt = @event.OccurredAt
        });
    }
}
```

---

### 6. 未決定事項の優先度 ⚠️ 高優先度 ✅ 反映済み

以下は実装前に確定すべき重要事項:

#### 高優先度（実装開始前に必須）

| 項目 | 現状 | 推奨 | 理由 |
|------|------|------|------|
| **マルチテナント分離方式** | 未定 | シングルDB + TenantId | シンプルで実装コストが低い。行レベルでテナント分離 |
| **JWT方式** | 未定 | AccessToken（15分）+ RefreshToken（7日）ローテーション | セキュリティとUXのバランス |
| **ログイン失敗ロック閾値** | 未定 | 5回失敗で30分ロック | セキュリティポリシーとして明確化 |

#### 中優先度（初期実装中に確定）

| 項目 | 現状 | 推奨 |
|------|------|------|
| DB | PostgreSQL推奨 | PostgreSQL |
| パスワードポリシー | 未定 | 最低8文字、大小英数字+記号 |
| RefreshToken有効期限 | 未定 | 7日間 |

---

## 追加で検討すべき項目 ✅ 反映済み（一部）

### セキュリティ

| # | 項目 | 現状 | 推奨 | 対応 |
|---|------|------|------|------|
| S-01 | **CSRF対策** | 未定義 | RefreshToken使用時のトークンバインディング実装 | ✅ 仕様に追加 |
| S-02 | **XSS対策** | 未定義 | JWT保存場所をHttpOnly Cookieに（LocalStorageは脆弱） | ✅ 仕様に追加 |
| S-03 | **パスワードリセット** | 未定義 | メール認証フロー + 一時トークン発行 | ✅ ユースケースA-04/A-05・フロー図を追加 |
| S-04 | **メール検証** | 未定義 | ユーザー招待時の検証フロー（招待リンク有効期限） | ✅ ユースケースA-06を追加 |
| S-05 | **2要素認証（2FA）** | 未定義 | 将来対応を考慮した設計（TOTP推奨） | ⏭️ 将来検討項目として維持 |
| S-06 | **レート制限** | 未定義 | ログインAPI: 5回/分、トークンリフレッシュ: 10回/分 | ✅ セキュリティ考慮事項セクションに追加 |

### 運用

| # | 項目 | 現状 | 推奨 | 対応 |
|---|------|------|------|------|
| O-01 | **RefreshTokenクリーンアップ** | 未定義 | 期限切れトークンの定期削除（日次バッチ） | ✅ 運用考慮事項セクションに追加 |
| O-02 | **監査ログ保持期間** | 未定義 | 1年間保持 + アーカイブ戦略 | ✅ 運用考慮事項セクションに追加 |
| O-03 | **テナント削除** | 論理削除のみ | 論理削除後90日でデータ完全削除 | ✅ 運用考慮事項セクションに追加（方針は保持ポリシーで別途決定） |
| O-04 | **セッション管理** | 未定義 | 同時ログイン数制限（例：5デバイスまで） | ✅ 運用考慮事項セクションに追加（初期は制限なし） |

### パフォーマンス

| # | 項目 | 懸念 | 推奨 | 対応 |
|---|------|------|------|------|
| P-01 | **N+1問題** | User取得時のRoleロード | Eager Loading（`.Include(u => u.Roles)`） | ✅ パフォーマンス考慮事項に追加 |
| P-02 | **監査ログ書き込み** | 同期処理による遅延 | 非同期処理（バックグラウンドキュー） | ✅ パフォーマンス考慮事項に追加 |
| P-03 | **RefreshToken検索** | HashedTokenでの検索 | `HashedToken`カラムにインデックス追加 | ✅ 推奨インデックスに追加 |
| P-04 | **監査ログクエリ** | 大量データでの検索 | `(TenantId, OccurredAt)`複合インデックス | ✅ 推奨インデックスに追加 |

---

## データベース設計の補足

### 推奨インデックス

```sql
-- Tenant
CREATE INDEX idx_tenant_status ON Tenant(Status);

-- ApplicationUser
CREATE INDEX idx_user_tenant ON ApplicationUser(TenantId);
CREATE INDEX idx_user_email ON ApplicationUser(NormalizedEmail);

-- RefreshToken
CREATE UNIQUE INDEX idx_refreshtoken_hashed ON RefreshToken(HashedToken);
CREATE INDEX idx_refreshtoken_user ON RefreshToken(UserId);
CREATE INDEX idx_refreshtoken_expiry ON RefreshToken(ExpiresAt) WHERE Revoked = false;

-- AuditLog
CREATE INDEX idx_auditlog_tenant_time ON AuditLog(TenantId, OccurredAt DESC);
CREATE INDEX idx_auditlog_user ON AuditLog(UserId);
CREATE INDEX idx_auditlog_eventtype ON AuditLog(EventType);
```

### マルチテナント分離方式

**確定**: シングルDB + TenantId による行レベル分離

---

## ユースケースの追加提案 ✅ 反映済み

現在のユースケース一覧に以下を追加することを推奨:

### 認証（追加）
| # | ユースケース | 実行者 | 優先度 |
|---|------------|--------|--------|
| A-04 | パスワードリセット要求（メール送信） | 未認証ユーザー | 高 |
| A-05 | パスワードリセット実行（トークン検証） | 未認証ユーザー | 高 |
| A-06 | メールアドレス検証 | 未認証ユーザー | 中 |
| A-07 | 全デバイスログアウト | 認証済みユーザー | 中 |

### ユーザー管理（追加）
| # | ユースケース | 実行者 | 優先度 |
|---|------------|--------|--------|
| U-04 | ユーザー一覧取得（テナント内） | TenantAdmin | 高 |
| U-05 | ユーザー削除（論理削除） | TenantAdmin | 中 |
| U-06 | ロール変更 | TenantAdmin | 中 |
| U-07 | アカウントロック解除 | TenantAdmin | 低 |

### テナント管理（追加）
| # | ユースケース | 実行者 | 優先度 |
|---|------------|--------|--------|
| T-02 | テナント情報更新 | TenantAdmin | 中 |
| T-03 | テナント停止 | システム管理者 | 低 |
| T-04 | テナント削除（論理削除） | システム管理者 | 低 |

---

## 推奨される次のステップ

### フェーズ1: 設計修正（1-2週間）

1. ✅ ドメイン層のIdentity依存を解消する設計変更
2. ✅ 未決定事項（特にマルチテナント分離方式）の確定
3. ✅ セキュリティ要件の詳細化
4. ✅ ユースケースの追加・詳細化

### フェーズ2: 詳細設計（1-2週間）

1. データベーススキーマ設計
2. API仕様書作成（OpenAPI/Swagger）
3. エラーハンドリング戦略
4. ログ設計（アプリケーションログ vs 監査ログ）

### フェーズ3: 実装準備（1週間）

1. 実装タスクへの分解（Spec形式での構造化）
2. 開発環境セットアップ
3. CI/CDパイプライン設計

---

## まとめ

この設計は全体として良好ですが、以下の点を修正することで、よりDDD原則に忠実で保守性の高い実装が可能になります:

### 必須修正項目（実装前）
1. ドメイン層のIdentity依存解消
2. ログインフローのセキュリティ改善
3. マルチテナント分離方式の確定

### 推奨修正項目（初期実装中）
1. RefreshTokenの集約設計見直し
2. テナント登録の整合性戦略
3. AuditLogのイベント駆動設計

### 将来検討項目
1. 2要素認証対応
2. 監査ログの別DB移行
3. スケーラビリティ対策

この設計をベースに実装を進める場合、Kiroのspec形式（requirements.md、design.md、tasks.md）に変換して、段階的に実装を進めることをお勧めします。

---

**レビュアー**: Kiro AI Assistant  
**次回レビュー推奨時期**: 設計修正後
