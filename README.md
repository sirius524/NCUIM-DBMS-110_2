# DB Final Project Readme
NCU-IM2 2022 Spring DBMS Final Project


## 前情提要
### Naming Conventions
參考了問題整理->其他項目第二點，以下修改皆遵守不更動 input 參數內容，符合規範
我按照 mysql 官方的 guideline 修改了一些 input 欄位名稱
https://dev.mysql.com/doc/internals/en/coding-style.html
https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_NAMING_CONVENTIONS.html
guideline 有提到應用 snake case 命名而非 camel case，對於輸入欄位我都加一個 in 做 prefix，增加可讀性
![](https://i.imgur.com/9UXtSss.png)
#### Correction 如下
**純粹Naming**
- Proposal -> proposal
- ProposalMember -> proposal_member
- ProposalOption -> proposal_option
- SponsorRecord -> sponsor_record
- Category -> category
- FAQ -> faq
- Member -> member
    - member_id -> id

**欄位資料更動**
- Comment -> comment
    - created_at -> **Added**
這個欄位 SPEC 上沒有，我新增的
- FollowingRecord -> following_record
    - id -> **Removed**
根據正規化的原則，已經有其他的 unique 資料做檢索，因此我將它移除
- MemberCredential -> member_credential
    - id -> **Removed**
    - hashed_user_id -> member_id
同上，根據正規化的原則，id 直接取用上層 member 的，hashed 只需紀錄 passwd 就好


## StoreProcudure
- 2、3、4. sp_RegisterMember, sp_UpdatePwd, sp_Login
    - 根據當初作業的設計，為確保安全性我選擇 SHA-512 作為 hash funciton， hashed 完的 passwd 長度為 512 bits，放到 varchar(64) 就剛好了(64 x 8) varchar(200) 浪費執行階段的資源，因此改為 varchar(64)
- 2、4. sp_RegisterMember, sp_Login
    - 根據 RFC2821 指出，目前網路規範的 mail address max length 應考慮為 254 個 char，因此更該 varchar(64) to varchar(254) 用以滿足特定情況
    https://7php.com/the-maximum-length-limit-of-an-email-address-is-254-not-320/
- 10.sp_CreateProposal
    - 假資料 create_date 是 date，但是 result 有時分秒，因此改為 TimeStamp
## 疑義
### sp_GetFollowedProposalsByMember 
sponsor_record 跟 proposal 的 amount 重複了，假資料內的資料不確定是否有驗證過（估計加起來是不一樣），且根據正規化的原則，理論上不需要有 proposal 的 amount 欄位，應該去拉 sponsor_record 的來進行相加就好，我選擇以 sponsor_record 作為基準（複雜很多），但同時也實作了以 proposal table 的方法
```sql=
select
    proposal_member.member_id,
    proposal.title as proposal_title,
    proposal.amount as proposal_amount,
    proposal.goal as proposal_goal
from proposal
join proposal_member
on proposal.id = proposal_member.proposal_id
where proposal_member.member_id = in_member_id;
```
## 其他
#### sp_RegisterMember
透過 transaction funciton 設定斷點，確保兩個斷點的操作會成功，如果失敗不會有不完全的資料寫入 DB
#### sp_DeleteMember
在實作時有考慮過加 deactive 的欄位，但是需要修改的 store procedure 實在太多，期末在燒阿><
因此就直接將使用者刪除，child record 不重要的直接 cascade，重要的就 set null，但最好還是不要立即刪掉，應該有 deactive 欄位以及 delete_date maybe 30 天後再做刪除