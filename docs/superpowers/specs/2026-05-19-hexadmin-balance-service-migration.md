# HexAdmin: Migrate Balance Data from EAM to Balance Service

**Date:** 2026-05-19
**Status:** Draft
**Author:** rock.zeng

---

## Background

HexAdmin UI fetches balance-related data (enterprise account views, asset views, safe account asset details, asset distributions) via HexAdmin API, which in turn calls the Enterprise Account Master (EAM) service. The Balance Service (`hexsafe-2-balance-service`) was introduced as the authoritative source for balance data, and the HexSafe client UI already completed this migration in February 2026.

The goal of this change is to make HexAdmin consistent: balance data fetched via HexAdmin API should come from the Balance Service, while EAM remains the source of truth for enterprise/vault/safe account entity management only.

---

## Current State

### Two parallel implementations already exist in hexadmin-api

When the Balance Service was introduced, new handlers were added to hexadmin-api with the `admin_Bs_*` JRPC prefix that correctly call the Balance Service. However, the **original `admin_Enterprise_*` and `admin_Account_*` handlers were never removed** and still call EAM for balance data. This means:

- **hexadmin-ui `src/services/balance.ts`** calls `admin_Bs_*` → Balance Service ✅
- **hexadmin-ui `src/services/enterprise-account.ts`** calls `admin_Enterprise_*` and `admin_Account_*` → EAM ❌

Both sets of handlers serve similar data, but from different sources.

### EAM balance methods still called by hexadmin-api

| hexadmin-api handler (JRPC method) | EAM method | File |
|---|---|---|
| `admin_Enterprise_FetchEnterprise` | `hs2_eam_GetEnterpriseAccountView` | `handler/enterprise/handler/fetch_enterprise.go:152` |
| `admin_Enterprise_ListEnterpriseAssets` | `hs2_eam_GetEnterpriseAssetView` | `handler/enterprise/handler/list_enterprise_assets.go:198` |
| `admin_Enterprise_GetEnterpriseAssetDistribution` | `hs2_eam_GetAssetDistribution` | `handler/enterprise/handler/get_enterprise_asset_distribution.go:75` |
| `admin_Account_ListAccountAssets` | `hs2_eam_GetSafeAccountAssetDetails` | `handler/account/handler/list_account_assets.go:171` |

### Balance Service admin endpoints (already exist, fully implemented)

| Balance Service method | Handler file |
|---|---|
| `hs_admin_bs_GetEnterpriseAccountView` | `internal/service/balance_admin/getEnterpriseAccountView_admin.go` |
| `hs_admin_bs_GetEnterpriseAssetView` | `internal/service/balance_admin/getEnterpriseAssetView_admin.go` |
| `hs_admin_bs_GetSafeAccountAssetDetails` | `internal/service/balance_admin/getSafeAccountAssetDetails_admin.go` |
| `hs_admin_bs_GetAssetDistributions` | `internal/service/balance_admin/getAssetDistributions_admin.go` |

---

## Scope

### In scope

- Switch the 4 legacy hexadmin-api handlers from EAM to Balance Service
- Validate response schema compatibility; update response mapping if needed
- Ensure hexadmin-ui requires no code changes (same JRPC method names, same response shapes)

### Out of scope

- Staking balance data (`GetTotalStakedBalances`, `GetStakedBalances`, `GetClaimRewards`) — separate review needed
- NFT asset views — Balance Service has no NFT equivalent; EAM stays
- HTM service callers (`htm-htm-funding-engine`, `htm-rfq-engine`) — trading balance context; separate task
- Fee calculation engine EAM calls — separate review
- Cleanup of the now-redundant `admin_Bs_*` duplicate handlers in hexadmin-api (follow-up)

---

## Required Changes

### 1. HexAdmin API — 4 handler updates

Each handler needs to swap its downstream client from `internal/pkg/eam/` to `internal/pkg/balance_service/`.

#### `fetch_enterprise.go` — enterprise account view

- Remove call to `eam.GetEnterpriseAccountView()` (`internal/pkg/eam/enterprise.go:221`)
- Call `balance_service.GetEnterpriseAccountView()` (`internal/pkg/balance_service/bs_get_enterprise_account_view.go`)
- Verify response mapping: balance service returns per-account balance breakdown with fields `totalBalanceInBaseCurrency`, `availableBalanceInBaseCurrency`, `frozenBalanceInBaseCurrency`, `lockedBalanceInBaseCurrency`, `stakedBalanceInBaseCurrency`, `bondedBalanceInBaseCurrency`, etc.

#### `list_enterprise_assets.go` — enterprise asset view by ticker

- Remove call to `eam.GetEnterpriseAssetView()` (`internal/pkg/eam/enterprise.go:290`)
- Call `balance_service.GetEnterpriseAssetView()`
- Verify response mapping: balance service returns `AssetTickerInfo` with `assetTicker`, `assetTickerGroup`, `balanceDecimal`, `balanceInBaseCurrency`, and per-chain `assetNetworkInfoList`

#### `get_enterprise_asset_distribution.go` — asset distribution

- Remove call to `eam.GetAssetDistribution()` (`internal/pkg/eam/enterprise.go:428`)
- Call `balance_service.GetAssetDistributions()` (note: balance service version is paginated and named plural)
- **Requires schema validation:** balance service returns paginated response with `pageNumber`, `pageSize`, `totalItems`, `totalPages`, `sort`, and `sumOfBalances` aggregate. Map to existing API contract.

#### `list_account_assets.go` — safe account asset details

- Remove call to `eam.GetSafeAccountAssetDetails()` (`internal/pkg/eam/account.go:314`)
- Call `balance_service.GetSafeAccountAssetDetails()`
- Verify response mapping: balance service returns `AssetTickerInfoAggregated` with address-level breakdown

### 2. HexAdmin UI — no changes required (Option A)

The 4 legacy JRPC methods (`admin_Enterprise_FetchEnterprise`, `admin_Enterprise_ListEnterpriseAssets`, `admin_Enterprise_GetEnterpriseAssetDistribution`, `admin_Account_ListAccountAssets`) keep the same names. The UI continues to call them unchanged. Only the backend data source switches.

If response schema differences surface during testing, minimal changes to `src/services/enterprise-account.ts` and `src/namespaces/enterprise-account.ts` will be needed.

### 3. Balance Service — no new endpoints required

All 4 required admin endpoints already exist. No changes needed.

### 4. Env var — already configured in hexadmin-api

- EAM: `ENTERPRISE_ACCOUNT_MASTER_URL` (`internal/pkg/eam/config.go`)
- Balance Service: `BALANCE_SERVICE_URL` (default: `http://balance-service:8082`) (`internal/pkg/balance_service/config.go`)

No new infrastructure changes needed.

---

## EAM Methods That Stay (do not migrate)

| Category | EAM methods |
|---|---|
| Enterprise CRUD | `hs2_admin_eam_GetEnterprise/GetEnterprises/SearchEnterprise/CreateEnterprise/UpdateEnterpriseStatus/InitiateEnterpriseUpdate/ApproveEnterpriseUpdate/RejectEnterpriseUpdate` |
| Safe Account CRUD | `hs2_admin_eam_GetSafeAccount/GetSafeAccounts/CreateSafeAccount/UpdateSafeAccountStatus/InitiateSafeAccountUpdate/ApproveSafeAccountUpdate/RejectSafeAccountUpdate` |
| Custodian CRUD | `hs2_admin_eam_GetCustodian/GetCustodians/CreateCustodian/UpdateCustodian/UpdateCustodianStatus` |
| NFT views | `hs2_eam_GetEnterpriseNFTAssetView`, `hs2_eam_GetSafeAccountNFTAssetDetails`, `hs2_eam_GetNFTAssetDistribution` |
| Wallet addresses | `hs2_eam_GetWalletAddressDetails`, `hs2_eam_GetSafeAccountAddressesByChainId` |
| Trading accounts | `hs2_eam_GetTradingSafeAccounts` |
| Kafka republish ops | `hs2_admin_eam_PublishCustodiansToKafka`, `hs2_admin_eam_PublishEnterprisesToKafka`, `hs2_admin_eam_PublishSafeAccountsToKafka` |

---

## Other Callers Still Using EAM for Balances

These repos are outside hexadmin scope but should be tracked for future migration:

| Repo | EAM balance methods | Recommendation |
|---|---|---|
| `htm-htm-funding-engine` | `GetAssetDistribution`, `GetTradingAccountsView`, `GetTradingAccountAssetsDetail` | Review with HTM team — trading balance context may need separate admin balance endpoints |
| `htm-rfq-engine` | `GetAssetDistribution` | Review with HTM team |
| `hexsafe-2-fee-calculation-engine` | `hs2_admin_eam_*` account/enterprise calls | Separate task — AUM calculation scope |
| `htm-htm-account-service` | `hs2_admin_eam_*` account/custodian | Likely stays in EAM (entity management) |
| `gryfyn-qa` | Test specs for EAM balance APIs | Update QA specs after hexadmin migration is validated |

---

## Migration Order

1. **Schema diff** — Compare EAM and Balance Service response types for the 4 methods. Document any field-level differences before coding.
2. **hexadmin-api handler updates** — Switch the 4 handlers; add integration tests.
3. **Deploy to staging** — Verify hexadmin-ui shows correct data with no UI changes.
4. **Staking review** — Decide if `GetTotalStakedBalances`/`GetStakedBalances` in hexadmin-api also moves to Balance Service.
5. **Cleanup** — Remove dead EAM balance client code from `internal/pkg/eam/`; deprecate redundant `admin_Bs_*` handlers or migrate UI to use them directly (follow-up PR).
6. **HTM repos** — Separate task with HTM team.
7. **QA** — Update `gryfyn-qa` test specs.

---

## Open Questions

1. **Staking balance**: Should `hs2_admin_eam_GetTotalStakedBalances` and `hs2_eam_GetStakedBalances` also be migrated? The Balance Service tracks staked balances in `balances_current` and has `bk-staked-balance-btc` Kafka consumption. Needs confirmation whether admin staking balance view is covered.
2. **HTM trading balances**: `htm-htm-funding-engine` calls `GetAssetDistribution` against EAM. Is this trading-vault balance (separate) or shared balance data? If shared, the Balance Service `GetAssetDistributions` admin endpoint should cover it.
3. **Response schema delta**: Full diff of EAM vs Balance Service response types for `GetEnterpriseAssetView` — specifically `withdrawableBalance` vs `availableBalance` field naming needs to be confirmed consistent before removing EAM fallback.
