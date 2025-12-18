Excellent question! Let me trace the **EXACT path** from the Kairos webhook to the method execution. I'll show you the configuration and routing.

## **The Complete Connection Path**

### **STEP 1: Kairos Configuration (config/kairos.yml)**

[1 tool called]

Perfect! Now I can show you the complete flow. Let me trace it step by step:

## **The Complete Flow Explained**

### **STEP 1: Kairos Scheduler Setup (One-time configuration)**

In `config/kairos.yml`, the webhook URL is configured:

```57:59:config/kairos.yml
      purge_suspended_accounts:
        job_id: FS_PURGE_SUSPENDED_ACCOUNTS
        webhook_url: localhost.freshservice-dev.com:3000/fdadmin/scheduled_jobs/clock_works
```

Notice: The webhook URL points to `/fdadmin/scheduled_jobs/clock_works` (NOT a specific method!)

### **STEP 2: Rake Task Creates the Scheduled Job**

When you run `rake kairos_scheduler:purge_suspended_accounts`, this code executes:

```247:257:lib/tasks/kairos_scheduler.rake
	task :purge_suspended_accounts => :load_config do
		begin
			url_params = {"to_invoke"=>"purge_suspended_accounts"}
			config[:digest] = generate_kairos_md5_digest(url_params)

			payload={'group'=> config[:group], 'job_id'=> KairosConfig['jobs']['purge_suspended_accounts']['job_id'], 'data'=> {}, 'scheduled_time'=> "#{Time.now.utc.iso8601}" , 'recurring'=> { 'cron' => "30 12 * * 6" } ,'webhook'=> {:url => "#{KairosConfig['jobs']['purge_suspended_accounts']['webhook_url']}?digest=#{config[:digest]}&to_invoke=purge_suspended_accounts"}}
			response = RestClient::Request.execute(method: :post, url: config[:url], headers: config[:header], payload: payload.to_json)
			puts response
		rescue => e
			puts "Error while scheduling job in kairos for purge_suspended_accounts #{e.message}"
			NewRelic::Agent.notice_error(e,{:description => "Error while scheduling job in kairos for purge_suspended_accounts"})
```

**Key Point**: Look at line 252 carefully! The webhook URL is constructed as:

```
#{KairosConfig['jobs']['purge_suspended_accounts']['webhook_url']}?digest=#{config[:digest]}&to_invoke=purge_suspended_accounts
```

Which becomes:
```
localhost.freshservice-dev.com:3000/fdadmin/scheduled_jobs/clock_works?digest=XXXXX&to_invoke=purge_suspended_accounts
```

The **query parameter** `to_invoke=purge_suspended_accounts` is added to the URL!

### **STEP 3: Kairos Service Stores This Configuration**

The payload is sent to Kairos scheduler service with:
- `cron: "30 12 * * 6"` (every Saturday at 12:30 PM UTC)
- `webhook.url: "...clock_works?to_invoke=purge_suspended_accounts"`

Kairos stores this and will trigger it every Saturday.

### **STEP 4: Every Saturday, Kairos Makes HTTP Request**

When the cron time arrives, Kairos makes an HTTP POST request to:

```
POST https://your-freshservice-domain.com/fdadmin/scheduled_jobs/clock_works?digest=XXXXX&to_invoke=purge_suspended_accounts
```

### **STEP 5: Rails Routing**

Let me show you the route definition:

[2 tools called]

Perfect! Now I can show you the complete path. Here's the route definition:

```1981:1992:config/routes.rb
      resources :scheduled_jobs,:only => :index do
        collection do
          post :destroy_feature_data
          post :task_remainders
          post :journey_task_unassigned
          post :journey_task_reminder
          post :multi_account_msp_digest
          post :clear_central_backup_queue
          post :purge_spam_and_deleted_tickets
          post :archive_tickets
          post :purge_deleted_pcr
          post :clock_works
```

This creates a route: `POST /fdadmin/scheduled_jobs/clock_works`

### **STEP 6: Controller Receives the Request**

When Kairos calls the webhook URL with `?to_invoke=purge_suspended_accounts`, Rails routes it to:

```169:172:app/controllers/fdadmin/scheduled_jobs_controller.rb
  def clock_works
    safe_send(params[:to_invoke])
    head :ok
  end
```

**The Magic**: The `params[:to_invoke]` contains `"purge_suspended_accounts"` from the query parameter!

So `safe_send(params[:to_invoke])` becomes `safe_send("purge_suspended_accounts")`

### **STEP 7: safe_send Dynamically Calls the Method**

`safe_send` is a Ruby method that **dynamically calls** another method by name. So:

```ruby
safe_send("purge_suspended_accounts")
```

Is equivalent to:

```ruby
self.purge_suspended_accounts
```

Which calls this method in the SAME controller:

```664:666:app/controllers/fdadmin/scheduled_jobs_controller.rb
    def purge_suspended_accounts
      GlobalSidekiqWorker.enqueue(PurgeSuspendedAccounts, {})
    end
```

## **Visual Flow with Code References**

```
┌─────────────────────────────────────────────────────────────────────┐
│ ONE-TIME SETUP (Rake Task)                                         │
│ lib/tasks/kairos_scheduler.rake:247                                 │
├─────────────────────────────────────────────────────────────────────┤
│ rake kairos_scheduler:purge_suspended_accounts                      │
│                                                                     │
│ Creates webhook in Kairos:                                         │
│ URL: clock_works?to_invoke=purge_suspended_accounts                │
│ Cron: "30 12 * * 6" (Saturday 12:30 PM UTC)                        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 │ Kairos stores this configuration
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│ EVERY SATURDAY AT 12:30 PM UTC                                      │
│ Kairos Scheduler Service                                            │
├─────────────────────────────────────────────────────────────────────┤
│ Kairos makes HTTP POST request to:                                 │
│                                                                     │
│ POST /fdadmin/scheduled_jobs/clock_works?                          │
│      digest=XXXXX&                                                  │
│      to_invoke=purge_suspended_accounts                            │
│                                                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ HTTP POST
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│ RAILS ROUTING                                                       │
│ config/routes.rb:1992                                               │
├─────────────────────────────────────────────────────────────────────┤
│ POST /fdadmin/scheduled_jobs/clock_works                           │
│                                                                     │
│ Routes to:                                                          │
│   controller: "fdadmin/scheduled_jobs"                             │
│   action: "clock_works"                                            │
│                                                                     │
│ params = {                                                          │
│   to_invoke: "purge_suspended_accounts",  ← from query param       │
│   digest: "XXXXX"                                                   │
│ }                                                                   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 │ Rails dispatches to controller
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│ CONTROLLER ACTION                                                   │
│ app/controllers/fdadmin/scheduled_jobs_controller.rb:169            │
├─────────────────────────────────────────────────────────────────────┤
│ def clock_works                                                     │
│   safe_send(params[:to_invoke])    ← params[:to_invoke] is         │
│   head :ok                              "purge_suspended_accounts"  │
│ end                                                                 │
│                                                                     │
│ safe_send("purge_suspended_accounts") calls ──────────┐            │
└───────────────────────────────────────────────────────┼────────────┘
                                                        │
                                                        │ method call
                                                        │
┌───────────────────────────────────────────────────────▼────────────┐
│ SAME CONTROLLER - PRIVATE METHOD                                   │
│ app/controllers/fdadmin/scheduled_jobs_controller.rb:664            │
├─────────────────────────────────────────────────────────────────────┤
│ def purge_suspended_accounts                                       │
│   GlobalSidekiqWorker.enqueue(PurgeSuspendedAccounts, {})         │
│ end                                                                 │
│                                                                     │
│ Enqueues Sidekiq job ─────────────┐                               │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │
                                    │ job enqueued
                                    │
┌───────────────────────────────────▼─────────────────────────────────┐
│ SIDEKIQ WORKER                                                      │
│ app/workers/purge_suspended_accounts.rb:5                           │
├─────────────────────────────────────────────────────────────────────┤
│ class PurgeSuspendedAccounts < BaseWorker                          │
│   def perform(args)                                                 │
│     # Find all suspended accounts > 90 days                        │
│     # Enqueue ClearAccountData for each account                    │
│   end                                                               │
│ end                                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## **The Key Connection: Query Parameters!**

The clever part is that **one webhook URL** (`clock_works`) can trigger **many different jobs** based on the `to_invoke` query parameter:

### **Example: Different Jobs Using Same Webhook**

```59:59:config/kairos.yml
        webhook_url: localhost.freshservice-dev.com:3000/fdadmin/scheduled_jobs/clock_works
```

Different rake tasks add different `to_invoke` parameters:

1. **Purge Suspended Accounts**:
   ```ruby
   url: "clock_works?to_invoke=purge_suspended_accounts"
   → calls purge_suspended_accounts method
   ```

2. **Destroy CMDB Data**:
   ```ruby
   url: "clock_works?to_invoke=destroy_cmdb_data"
   → calls destroy_cmdb_data method
   ```

3. **Hardware Asset Expiry Notification**:
   ```ruby
   url: "clock_works?to_invoke=hardware_asset_expiry_notification_by_timezone"
   → calls hardware_asset_expiry_notification_by_timezone method
   ```

All use the **same `clock_works` endpoint** but trigger **different methods** based on the `to_invoke` parameter!

## **Why This Design?**

This design allows:
1. **One webhook endpoint** handles multiple scheduled jobs
2. **Easy authentication** - same digest validation for all jobs
3. **Centralized routing** - all Kairos jobs go through `clock_works`
4. **Flexible job addition** - add new jobs without creating new endpoints

## **Summary: The Complete Chain**

```
Kairos Cron (Saturday 12:30 PM)
    ↓ (HTTP POST)
Rails Route: POST /fdadmin/scheduled_jobs/clock_works?to_invoke=purge_suspended_accounts
    ↓ (routing)
Controller: clock_works method
    ↓ (safe_send with params[:to_invoke])
Controller: purge_suspended_accounts method (same controller)
    ↓ (enqueue)
Sidekiq: PurgeSuspendedAccounts worker
    ↓ (enqueue multiple)
Sidekiq: ClearAccountData worker (one per account)
    ↓ (calls)
Model: perform_destroy method
    ↓ (enqueue multiple platform workers)
Sidekiq: Platform-specific deletion workers (AMS, OCS, WaaS, etc.)
```

Does this make the flow clearer now? The key is understanding that `clock_works` is a **dispatcher** that uses the `to_invoke` parameter to call the right method!
