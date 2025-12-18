Ah! You want to understand **when and how the rake task itself gets called**, and the **complete timeline** from setup to execution. Let me show you the COMPLETE flow from beginning to end!

## **Complete Timeline: Start to End**

### **PHASE 1: ONE-TIME SETUP (Manual/Deployment)**

This happens **ONCE** when setting up the scheduler or during deployment:

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: DevOps/Admin runs the rake task MANUALLY                   │
│ (This is done ONCE during setup/deployment)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ $ rake kairos_scheduler:purge_suspended_accounts                    │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ This executes code in:                                          │ │
│ │ lib/tasks/kairos_scheduler.rake:247-257                         │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Rake task reads config and prepares payload                │
│ lib/tasks/kairos_scheduler.rake:247-257                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ task :purge_suspended_accounts => :load_config do                  │
│   begin                                                             │
│     # Prepare URL parameters                                       │
│     url_params = {"to_invoke"=>"purge_suspended_accounts"}         │
│                                                                     │
│     # Generate security digest                                     │
│     config[:digest] = generate_kairos_md5_digest(url_params)       │
│                                                                     │
│     # Build payload for Kairos                                     │
│     payload = {                                                     │
│       'group' => config[:group],                      # Grouping   │
│       'job_id' => 'FS_PURGE_SUSPENDED_ACCOUNTS',     # Unique ID  │
│       'data' => {},                                                 │
│       'scheduled_time' => Time.now.utc.iso8601,      # Start time │
│       'recurring' => {                                              │
│         'cron' => "30 12 * * 6"    ← EVERY SATURDAY 12:30 PM UTC  │
│       },                                                            │
│       'webhook' => {                                                │
│         url: "localhost...clock_works?to_invoke=purge_suspended_accounts" │
│       }                                                             │
│     }                                                               │
│   end                                                               │
│ end                                                                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼ HTTP POST
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Rake task sends payload to Kairos Service                  │
│ lib/tasks/kairos_scheduler.rake:253                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ RestClient::Request.execute(                                       │
│   method: :post,                                                    │
│   url: "https://scheduler.freshworksapi.com/schedules/",          │
│   headers: config[:header],                                        │
│   payload: payload.to_json                                         │
│ )                                                                   │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Kairos Service STORES the scheduled job                    │
│ External Service: scheduler.freshworksapi.com                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Kairos stores in its database:                                     │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Job ID: FS_PURGE_SUSPENDED_ACCOUNTS                         │   │
│ │ Cron Schedule: "30 12 * * 6"  (Every Saturday 12:30 PM)    │   │
│ │ Webhook URL: .../clock_works?to_invoke=purge_suspended_acc  │   │
│ │ Status: ACTIVE                                               │   │
│ │ Next Run: <calculated next Saturday>                        │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ ✓ Setup Complete! Job is now scheduled.                           │
└─────────────────────────────────────────────────────────────────────┘
```

**Important**: The rake task **does NOT run every Saturday**. It runs **ONCE** to register the job with Kairos. After that, Kairos handles the scheduling.

---

### **PHASE 2: AUTOMATIC WEEKLY EXECUTION (Every Saturday)**

This happens **AUTOMATICALLY every Saturday at 12:30 PM UTC** without any manual intervention:

```
┌─────────────────────────────────────────────────────────────────────┐
│ EVERY SATURDAY AT 12:30 PM UTC                                      │
│ Kairos Service (scheduler.freshworksapi.com)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Kairos checks its database:                                        │
│ - Current time: Saturday 12:30 PM UTC                              │
│ - Finds job: FS_PURGE_SUSPENDED_ACCOUNTS                           │
│ - Cron matches: "30 12 * * 6" ✓                                   │
│                                                                     │
│ Kairos makes HTTP POST request:                                    │
│                                                                     │
│ POST https://your-account.freshservice.com/fdadmin/                │
│      scheduled_jobs/clock_works?                                   │
│      digest=ABC123&                                                 │
│      to_invoke=purge_suspended_accounts                            │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ HTTP POST (webhook callback)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ YOUR FRESHSERVICE APPLICATION                                       │
│ Rails receives HTTP request                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Request:                                                            │
│   POST /fdadmin/scheduled_jobs/clock_works                         │
│   Params: {                                                         │
│     digest: "ABC123",                                              │
│     to_invoke: "purge_suspended_accounts"                          │
│   }                                                                 │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼ Rails routing
┌─────────────────────────────────────────────────────────────────────┐
│ RAILS ROUTER                                                        │
│ config/routes.rb:1992                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Route matches: POST scheduled_jobs/clock_works                     │
│                                                                     │
│ Dispatches to:                                                      │
│   Controller: Fdadmin::ScheduledJobsController                     │
│   Action: clock_works                                              │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ CONTROLLER: clock_works action                                      │
│ app/controllers/fdadmin/scheduled_jobs_controller.rb:169            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ def clock_works                                                     │
│   safe_send(params[:to_invoke])  ← "purge_suspended_accounts"     │
│   head :ok                                                          │
│ end                                                                 │
│                                                                     │
│ This dynamically calls:                                             │
│   self.purge_suspended_accounts                                    │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ method call in same controller
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ CONTROLLER: purge_suspended_accounts method                         │
│ app/controllers/fdadmin/scheduled_jobs_controller.rb:664            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ def purge_suspended_accounts                                       │
│   GlobalSidekiqWorker.enqueue(PurgeSuspendedAccounts, {})         │
│ end                                                                 │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ enqueue to Sidekiq
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ SIDEKIQ: PurgeSuspendedAccounts worker                             │
│ app/workers/purge_suspended_accounts.rb:5                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ def perform(args)                                                   │
│   # Find accounts with cancelled_at <= 90.days.ago                │
│   # AND state = 'suspended'                                        │
│                                                                     │
│   Subscription.where(                                              │
│     "cancelled_at <= ? AND state = 'suspended'",                   │
│     90.days.ago                                                     │
│   ).each do |subscription|                                         │
│     # Enqueue ClearAccountData for each account                    │
│     GlobalSidekiqWorker.enqueue_at(                                │
│       index.hour.from_now,                                         │
│       ClearAccountData,                                            │
│       {account_id: subscription.account_id}                        │
│     )                                                               │
│   end                                                               │
│ end                                                                 │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ enqueues multiple jobs
                                 │ (1 per account, staggered)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ SIDEKIQ: Multiple ClearAccountData workers                         │
│ (One job per account, staggered by 1 hour each)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Job 1: ClearAccountData for account_id: 12345                      │
│   Scheduled: Saturday 12:30 PM                                     │
│                                                                     │
│ Job 2: ClearAccountData for account_id: 12346                      │
│   Scheduled: Saturday 1:30 PM                                      │
│                                                                     │
│ Job 3: ClearAccountData for account_id: 12347                      │
│   Scheduled: Saturday 2:30 PM                                      │
│                                                                     │
│ ... (one per account)                                              │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ each job runs at its scheduled time
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ SIDEKIQ: ClearAccountData worker (per account)                     │
│ app/workers/clear_account_data.rb:8                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ def perform(args)                                                   │
│   account = Account.find(args[:account_id])                        │
│                                                                     │
│   if subscription_cancel_complete?(account)                        │
│     send_account_destroy_payload(account)                          │
│     perform_destroy(account)  ← Main deletion logic               │
│   end                                                               │
│ end                                                                 │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ synchronous call
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODEL: perform_destroy method                                       │
│ lib/freshdesk_core/model.rb:252                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ def perform_destroy(account)                                       │
│   # Synchronous deletions                                          │
│   delete_jira_webhooks(account)                                    │
│   clear_attachments(account)                                       │
│   delete_data_from_tables(account)                                 │
│                                                                     │
│   # Async platform deletions (enqueues separate jobs)             │
│   remove_ams_data(account)         → Ams::AccountDeleter          │
│   remove_waas_data(account)        → WorkflowService::...         │
│   remove_status_page_data()        → StatusPage::AccountDeleter   │
│   delete_workato_account(account)  → Workato::AccountDeleter      │
│                                                                     │
│   # Main account deletion                                          │
│   account.destroy                                                   │
│                                                                     │
│   # Post-deletion cleanup                                          │
│   remove_ocs_data(account)         → Ocs::Worker::AccountDeleter  │
│ end                                                                 │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ multiple async jobs enqueued
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│ SIDEKIQ: Platform-specific deletion workers                        │
│ (All run independently in parallel)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ [Parallel Execution]                                                │
│                                                                     │
│ ├─ Ams::AccountDeleter                                             │
│ │  └─ DELETE account from AMS platform                            │
│ │                                                                   │
│ ├─ WorkflowService::AccountDeleter                                 │
│ │  └─ DELETE account from WaaS platform                           │
│ │                                                                   │
│ ├─ StatusPage::AccountDeleter                                      │
│ │  └─ DELETE account from Status Page platform                    │
│ │                                                                   │
│ ├─ Workato::AccountDeleter                                         │
│ │  └─ DELETE account from Workato platform                        │
│ │                                                                   │
│ └─ Ocs::Worker::AccountDeleter                                     │
│    └─ DELETE account from OCS platform                             │
│                                                                     │
│ ✓ Account deletion complete across all platforms                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## **Key Points Summary**

### **WHO CALLS WHAT:**

1. **DevOps/Admin** → Calls rake task (`rake kairos_scheduler:purge_suspended_accounts`)
   - **When**: During initial setup/deployment (ONE TIME)
   - **Purpose**: Register the job with Kairos

2. **Rake Task** → Sends HTTP POST to Kairos Service
   - **When**: Immediately when rake task runs
   - **Purpose**: Store the cron schedule in Kairos

3. **Kairos Service** → Calls webhook on YOUR Freshservice app
   - **When**: Every Saturday 12:30 PM UTC (AUTOMATICALLY)
   - **Purpose**: Trigger the actual deletion process

4. **Clock_works Controller** → Calls `purge_suspended_accounts` method
   - **When**: When webhook is received from Kairos
   - **Purpose**: Dispatch to the correct job based on `to_invoke` param

5. **purge_suspended_accounts method** → Enqueues `PurgeSuspendedAccounts` worker
   - **When**: Called by `clock_works`
   - **Purpose**: Start the async deletion process

6. **PurgeSuspendedAccounts worker** → Enqueues multiple `ClearAccountData` workers
   - **When**: Immediately after being enqueued
   - **Purpose**: Queue deletion for each eligible account

7. **ClearAccountData worker** → Calls `perform_destroy`
   - **When**: At staggered times (1 hour apart per account)
   - **Purpose**: Execute actual account deletion

8. **perform_destroy** → Enqueues platform-specific deleters
   - **When**: During ClearAccountData execution
   - **Purpose**: Delete account data from all platforms

### **IMPORTANT DISTINCTIONS:**

- **Rake task runs**: ONCE during setup
- **Kairos triggers webhook**: EVERY SATURDAY automatically
- **Account deletions**: Staggered throughout Saturday/Sunday

Does this complete timeline make the flow clear now?
