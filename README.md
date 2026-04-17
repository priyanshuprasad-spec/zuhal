# Zuhal Email Verifier - Architecture Overview

## Overview
Zuhal is a robust, distributed email validation application. The core logic of managing users, payments, tasks, and data lives in a **Laravel 10** web application. However, the intensive process of verifying emails is delegated to external server instances (such as Cloudways servers) using a specialized **Python script**.

This distributed architecture prevents the primary web application's IP from being blacklisted and allows horizontal scaling for massive verification tasks.

## System Architecture & Workflow

The architecture operates on a master-worker model, managed through SSH and SFTP:

1. **Task Initialization**: 
   A user uploads a list of emails to the Laravel web application. These emails are inserted into the central MySQL database (`email_lists` and `tasks` tables).

2. **Server Configuration**: 
   The application uses a pool of external servers configured via the Admin panel. These instances have Python 3 installed. The `FROM_EMAIL` references in `.env` (like `bnhgmfxbjx@1254975.cloudwaysapps.com`) are linked to these Cloudways worker servers used for SMTP handshakes.

3. **Delegating the Work via SSH and SFTP**:
   The external connections are explicitly triggered and managed by a specific set of Laravel classes using the `phpseclib3` package:
   - `app/Services/Admin/SSHService.php`: This is the core communication wrapper utilizing `phpseclib3\Net\SSH2` and `phpseclib3\Net\SFTP`. It exposes low-level methods like `executeSSHCommand()` and `executeSFTP()`.
   - `app/Http/Controllers/Admin/ServerDetailController.php`: When an admin configures a new worker server via the dashboard, this controller invokes `SSHService->executeSFTP()` to actively push the Python script onto the remote machine. It also runs pre-flight SSH checks (`test -d ~/venv/bin/` and `nc -zv`) to ensure a Python virtual environment exists and the worker server can communicate back to the master MySQL database.
   - `app/Services/RunPythonScriptService.php`: When a background email task begins, this service selects an available remote instance. It invokes `SSHService->executeSSHCommand()` to spawn the Python process asynchronously without waiting for it to finish, using:
     `nohup /usr/bin/python3 ~/path/email_validation_script.py <host> <user> <password> <database> <task_id> <appUrl> <statusFlag> > /dev/null 2>&1 &`

4. **Remote Execution (The Python Script)**:
   - The Python script running on the remote instance receives the central database credentials as command-line arguments.
   - It connects *back* to the master MySQL database to fetch the emails assigned to its `task_id`.
   - It performs highly concurrent checks (using `ThreadPoolExecutor`):
     - Syntax validation.
     - Disposable domain checking.
     - DNS MX resolution.
     - 'Accept-All' / Catch-all domain checking.
     - Active SMTP handshakes to verify mailboxes without sending emails.
   
5. **Real-time Updates**:
   - As the Python script verifies each email, it executes `UPDATE` queries directly against the master MySQL database to flag emails as `valid`, `invalid`, `catch all`, etc.
   - Upon completion, the script updates the task status to `completed` in the `tasks` table.

## Key Components

- **Frontend**: Laravel Blade / UI, augmented with Vite and potentially Pusher for real-time task progress updates.
- **Backend Framework**: Laravel 10 (PHP 8.1+).
- **Primary Database**: MySQL 8.
- **Verification Engine**: Python 3 (`email_validation_org.py`). Relies heavily on `dns.resolver`, `smtplib`, and `mysql-connector-python`.
- **Integrations**: Stripe/PayPal/Razorpay for billing, Google OAuth for authentication, Mailchimp, and Twilio.

## Why is it built this way?
Running email verification from the same web server hosting your application (the "same instance") is dangerous because SMTP connection tests trigger spam defenses from major inbox providers (Gmail, Outlook). This usually results in the server IP getting completely blacklisted. 
By delegating the verification to disposable/rotated Cloudways servers (the worker nodes), the system isolates the central web app's IP reputation from the harsh realities of verification SMTP pings.
