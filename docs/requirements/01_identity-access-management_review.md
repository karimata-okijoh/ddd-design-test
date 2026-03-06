# ID管理・ログイン管理（IAM）設計 レビュー

**レビュー日**: 2026-03-06  
**対象ドキュメント**: `01_identity-access-management.md`  
**ステータス**: 設計改善提案

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

### 1. User集約の設計上の問題 ⚠️ 高優先度

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

### 2. RefreshTokenの集約設計 ⚠️ 中優先度

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

### 3. テナント登録時の整合性 ⚠️ 高優先度

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

### 4. ログインフローのセキュリティ ⚠️ 高優先度

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

### 5. AuditLogの集約設計 ⚠️ 中優先度

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

### 6. 未決定事項の優先度 ⚠️ 高優先度

以下は実装前に確定すべき重要事項:

#### 高優先度（実装開始前に必須）

| 項目 | 現状 | 推奨 | 理由 |
|------|------|------|------|
| **マルチテナント分離方式** | 未定 | スキーマ分離 | アーキテクチャ全体に影響。DB設計・クエリ実装が大きく変わる |
| **JWT方式** | 未定 | AccessToken（15分）+ RefreshToken（7日）ローテーション | セキュリティとUXのバランス |
| **ログイン失敗ロック閾値** | 未定 | 5回失敗で30分ロック | セキュリティポリシーとして明確化 |

#### 中優先度（初期実装中に確定）

| 項目 | 現状 | 推奨 |
|------|------|------|
| DB | PostgreSQL推奨 | PostgreSQL（スキーマ分離対応） |
| パスワードポリシー | 未定 | 最低8文字、大小英数字+記号 |
| RefreshToken有効期限 | 未定 | 7日間 |

---

## 追加で検討すべき項目

### セキュリティ

| # | 項目 | 現状 | 推奨 |
|---|------|------|------|
| S-01 | **CSRF対策** | 未定義 | RefreshToken使用時のトークンバインディング実装 |
| S-02 | **XSS対策** | 未定義 | JWT保存場所をHttpOnly Cookieに（LocalStorageは脆弱） |
| S-03 | **パスワードリセット** | 未定義 | メール認証フロー + 一時トークン発行 |
| S-04 | **メール検証** | 未定義 | ユーザー招待時の検証フロー（招待リンク有効期限） |
| S-05 | **2要素認証（2FA）** | 未定義 | 将来対応を考慮した設計（TOTP推奨） |
| S-06 | **レート制限** | 未定義 | ログインAPI: 5回/分、トークンリフレッシュ: 10回/分 |

### 運用

| # | 項目 | 現状 | 推奨 |
|---|------|------|------|
| O-01 | **RefreshTokenクリーンアップ** | 未定義 | 期限切れトークンの定期削除（日次バッチ） |
| O-02 | **監査ログ保持期間** | 未定義 | 1年間保持 + アーカイブ戦略 |
| O-03 | **テナント削除** | 論理削除のみ | 論理削除後90日でデータ完全削除 |
| O-04 | **セッション管理** | 未定義 | 同時ログイン数制限（例：5デバイスまで） |

### パフォーマンス

| # | 項目 | 懸念 | 推奨 |
|---|------|------|------|
| P-01 | **N+1問題** | User取得時のRoleロード | Eager Loading（`.Include(u => u.Roles)`） |
| P-02 | **監査ログ書き込み** | 同期処理による遅延 | 非同期処理（バックグラウンドキュー） |
| P-03 | **RefreshToken検索** | HashedTokenでの検索 | `HashedToken`カラムにインデックス追加 |
| P-04 | **監査ログクエリ** | 大量データでの検索 | `(TenantId, OccurredAt)`複合インデックス |

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

### マルチテナント分離方式の比較

| 方式 | メリット | デメリット | 推奨度 |
|------|---------|-----------|--------|
| **シングルDB + TenantId** | シンプル、コスト低 | データ漏洩リスク高、スケール困難 | ❌ 非推奨 |
| **スキーマ分離** | セキュリティ向上、スケール可能 | 複雑度中、マイグレーション管理 | ✅ 推奨 |
| **DB分離** | 最高セキュリティ、独立スケール | 複雑度高、コスト高 | △ エンタープライズ向け |

**推奨**: スキーマ分離方式

```csharp
// 実装例
public class TenantDbContext : IdentityDbContext<ApplicationUser>
{
    private readonly string _tenantSchema;
    
    public TenantDbContext(string tenantSchema)
    {
        _tenantSchema = tenantSchema;
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.HasDefaultSchema(_tenantSchema);
        // ...
    }
}
```

---

## ユースケースの追加提案

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
