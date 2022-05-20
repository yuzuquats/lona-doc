- **Primary**: Application + Sqlite
- **Secondary (Backup)**: Application + Sqlite
- **Beta:** Application + Sqlite
  - All DB transactions are relayed to either primary/secondary
- **Deveopment**: Application + Sqlite



[[Uptime flow]]:
1. **S3** contains a lock file indicating if **Primary** or **Secondary** is the active machine
  - `.active-server-lock`
  ```
    {
      expiration: UTC time (S3 server clock),
      machine: primary | secondary
    }
  ```
2. **Secondary** messages **Primary** with a HEART_BEAT message
   2a. (HTTP Success) **Primary** responds with the current lease: `.active-server-lock`
   2b. (HTTP Failure) [[Failure Flow]]
3. **Primary** messages **S3** with an .active-server-lock request every 10 seconds
4. **Primary** syncs sqlite to **S3 Bucket** + **Secondary**
  - S3: primary/db.sqlite → s3/db.primary.sqlite
  - secondary: primary/db.sqlite → secondary/db.from-primary.sqlite



[[Shutdown Flow]]:
- **Primary** turns off gracefully
  1. Fails incoming HTTP requests
  2. Messages **S3** to delete the `.active-server-lock`
  3. Messages the CDN to stop all-incoming requests (?)
  4. Sends a RELEASE message to **Secondary** indicating that it’s shutting down
     3a. (HTTP Success) [[Startup Flow]]
     3b. (HTTP Failure) [[Failure Flow]]



[[Startup Flow]]:
- **Secondary** (or **Primary**)
  1. Performs a copy of DB
    - secondary: secondary.from-primary.sqlite (r) → secondary/db.sqlite (rw)
  2. Asks the CDN to begin redirecting traffic
  3. Begins receiving traffic
  4. Syncs sqlite to **S3**
    - S3: secondary/db.sqlite → s3/db.secondary.sqlite



[[Failure Flow]]:
- **Secondary** (or **Primary**)
  1. Acquires `.active-server-lock` from **S3**
     1a. If not gone and not expired
        - Saves the `.active-server-lock`
        - Waits 12s and rechecks the **S3** lock
          - If `.active-server-lock` changed
            - [[Split Brain]]
        - Messages **Primary** another HEART_BEAT message
          - If HTTP Success
            - [[Uptime Flow]]  (false alarm? nothing looks wrong)
        - Confirms with CDN that no servers are pointed to
  2. Acquires the `.active-server-lock` from **S3**
  3. [[Startup Flow]]



[[Deployment Flow]]:
1. Shutdown **Primary**: [[Shutdown Flow]]
  - (**Secondary** will automatically spin up)
2. Deploy code changes to **Primary**
3. Test **Primary** with production DB
  - DB writes don't affect **Secondary**
    - S3: primary/db.sqlite → s3/db.primary.sqlite
    - secondary: primary/db.sqlite → secondary/db.from-primary.sqlite
4. Shutdown **Secondary** [[Shutdown Flow]]



[[Split Brain]]:
- Shit out of luck
- Contact developer to restart both servers and debug



[[Beta API Request Flow]]
1. **Beta** does not sync sqlite to **S3 Bucket** + **Primary/Secondary**
2. **Beta** relays DB requests to `.active-server-lock["machine"]`