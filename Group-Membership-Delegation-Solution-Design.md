# Solution Design – Delegated Group Membership Administration

**Goal:** Designated business users maintain membership of specific ServiceNow groups, without IT or Entra ID involvement.
**Approach:** Use OOTB group / group-member tables, add one flag + one user-list field on the group, one role, and a few ACLs. No new tables, no UI build.

---

## 1. Prerequisites / Decisions

- [ ] Delegated groups must be **ServiceNow-local** groups (not synced from Entra ID/LDAP). If a group is synced, the sync will overwrite manual changes (breaks AC5). Exclude them from the sync/provisioning scope.
- [ ] Get the list of groups to delegate and the people who will administer each one.

## 2. Build Steps (in a scoped app or one Update Set)

### Step 1 – Add two fields to `sys_user_group`
| Label | Name | Type | Default | Notes |
|---|---|---|---|---|
| Delegated administration | `u_delegated_admin` | True/False | false | Allow-list flag. Default false = every group is protected unless explicitly opened (AC7) |
| Group administrators | `u_group_admins` | List (ref → `sys_user`) | empty | Who may maintain members of *this* group (AC1, AC4) |

Add both to the Group form. Keep the existing write ACLs on `sys_user_group` untouched (admin / user_admin only), so Group Administrators cannot change these fields or the group itself.

### Step 2 – Create role
- Role: `group_membership_admin` (description: "Can maintain members of groups where they are listed as Group Administrator"). No contained roles.
- Assign it to the designated users (or to a helper group that holds the role). This only gives *access to the function*; scope is controlled in Step 4.

### Step 3 – Script Include `GroupAdminUtil` (client-callable not required)
```javascript
var GroupAdminUtil = Class.create();
GroupAdminUtil.prototype = {
    initialize: function() {},

    // true if current user is a listed admin of an opened (delegated) group
    canManage: function(groupSysId) {
        if (!groupSysId) return false;
        var grp = new GlideRecord('sys_user_group');
        if (!grp.get(groupSysId)) return false;
        return grp.getValue('u_delegated_admin') == '1' &&
               ('' + grp.getValue('u_group_admins')).split(',').indexOf(gs.getUserID()) > -1;
    },
    type: 'GroupAdminUtil'
};
```

### Step 4 – ACLs (Security > ACL)
| Table / field | Operation | Role | Condition / Script |
|---|---|---|---|
| `sys_user_grmember` | create | `group_membership_admin` | `answer = new GroupAdminUtil().canManage(current.getValue('group'));` |
| `sys_user_grmember` | delete | `group_membership_admin` | same script |
| `sys_user_grmember` | read | `group_membership_admin` | same script (so they only see members of their groups) |
| `sys_user_group` | read | `group_membership_admin` | `answer = new GroupAdminUtil().canManage(current.getValue('sys_id'));` |
| `sys_user` (needed to pick users) | read | – | Verify in test that the picker works; only add a read ACL for `group_membership_admin` on `sys_user` (fields name, email, user_name, active) if it is blocked |

Do **not** create write/update ACLs for this role on `sys_user_grmember`, `sys_user_group`, `sys_group_has_role` or `sys_user_has_role`. Default behaviour keeps those admin-only, which gives AC4 and AC7.

### Step 5 – Module for Group Administrators
Application Navigator > new Application Menu "Group Administration" > Module **My Groups**:
- Table: `sys_user_group`
- Filter: `u_delegated_admin = true` AND `u_group_admins` **contains** `javascript:gs.getUserID()`
- Role: `group_membership_admin`

They open the group and use the **Group Members** related list (New / Edit… / Delete).

### Step 6 – Auditing (AC6)
- Dictionary of `sys_user_grmember`: confirm **Audit** (attribute `audit=true`) is on. Added members are visible via the `sys_user_grmember` record (`sys_created_by` / `sys_created_on`) and the audit/history; removed members are stored in **System Definition > Deleted Records** (`sys_audit_delete`) with who and when.
- Optional (nice to have): turn on audit for `u_group_admins` and `u_delegated_admin` on `sys_user_group` to track who opened or delegated a group.
- Optional: ready-made report "Group membership changes" on `sys_audit` / `sys_audit_delete` for table `sys_user_grmember`.

### Step 7 – Configure the data (admin task, no code)
For each delegated group: set **Delegated administration = true** and fill **Group administrators**. Never set it on admin/system groups (e.g. `admin`, `security_admin`, roles-heavy platform groups). Optional guard: a Before Business Rule on `sys_user_group` that aborts if `u_delegated_admin` is true and the group (or its parent groups) holds `admin`/`security_admin`/`user_admin`.

## 3. Test Plan

| # | Test (impersonate) | Expected | AC |
|---|---|---|---|
| 1 | User without `group_membership_admin` opens group member list | Cannot add/remove | AC1 |
| 2 | Group Admin of group A adds an existing user to A | Succeeds; user appears in group assignment lists and approvals | AC1, AC2 |
| 3 | Group Admin removes that user | Succeeds; user no longer in assignment/approval lists | AC3 |
| 4 | Group Admin of A tries to open/add to group B (flag true, not listed) | Group not visible; create/delete denied | AC4 |
| 5 | Group Admin tries to modify the group record, its roles, or the admin list | Denied (read-only) | AC4, AC7 |
| 6 | Group Admin tries admin group / group with flag false | Not visible, no member changes possible | AC7 |
| 7 | Complete flow with no Entra/IT involvement | Works in ServiceNow only | AC5 |
| 8 | Check `sys_user_grmember` created-by/on and Deleted Records | Who/when identifiable | AC6 |

Note for testers: role inheritance for a user whose membership just changed is picked up on their next session/login; assignments and approvals are evaluated live.

## 4. Out of Scope / Caveats
- Group Administrators can add any existing active user, but cannot create users or change roles on the group.
- Security caveat: adding someone to a group that holds roles grants those roles. Therefore only open groups whose roles are acceptable for business-level delegation (admin decision in Step 7).
- Flow-based approval of membership changes was deliberately not added to keep this simple.
