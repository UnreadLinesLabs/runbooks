# Create a Broadcom account

This guide walks through creating a Broadcom Support Portal account, signing in, and verifying access by downloading a free VMware product. Screenshots are numbered in the order they were captured.

## 1. Scope and dependencies

This runbook creates the Broadcom Support Portal account that every VMware Workstation download in this
repository depends on, and proves it works by downloading a free product end to end.

It covers registration, e-mail verification, sign-in, the Trade Compliance form Broadcom shows before a
first download, and verifying the downloaded file's integrity. It does **not** install VMware
Workstation, license it, or configure anything —
`prepare-windows-ubuntu-templates/README.md` starts from a working installation.

There is no architecture section here: this runbook builds no infrastructure, it only obtains an account
on a third-party portal.

Broadcom reworks this portal from time to time. The screenshots show the flow as captured on the day of
recording; if a screen no longer matches, the **sequence of actions** is the part that stays reliable.

## 2. Prerequisites

- An e-mail address you can receive a verification code at.
- A web browser.
- Nothing else — this runbook starts from zero.

## 3. Start from the Broadcom homepage

Go to [broadcom.com](https://www.broadcom.com).

![Broadcom homepage](screenshots/01-broadcom-homepage.png)

## 4. Open the registration form

Click the **Support Portal** dropdown in the top navigation, then click **Register**.

![Support Portal menu with Register option](screenshots/02-support-portal-menu-register.png)

## 5. Enter your email and solve the captcha

On the User Registration page, enter your email address and complete the captcha challenge. Click **Next**.

![Registration email and captcha](screenshots/03-registration-email-captcha.png)

> Use a personal, non-shared email address — Broadcom's Terms of Use flag shared inboxes and distribution lists (PDLs) as a security risk.

## 6. Verify your email address

A verification code is sent to your inbox. Enter the 6-digit code and click **Verify & Continue**.

![Email verification code entry](screenshots/04-email-verification-code.png)

## 7. Complete your registration details

Fill in:
- First Name / Last Name
- Country
- Job Title (optional)
- Password / Confirm Password

Accept the Terms of Use and Privacy Policy, then click **Create Account**.

![Registration form with personal details](screenshots/05-registration-form-details.png)

## 8. Confirm the account was created

The confirmation screen lists the services your new account has immediate access to (Product Documentation, Communities, Public Education, etc.).

![Registration success with service list](screenshots/06-registration-success-services.png)

## 9. Optionally build your profile

Scroll down to unlock additional services (Software Support Systems, VMware Hands-on Labs, Broadcom Partner, etc.) by building your profile. Choose **Yes, I want to Build my Profile** or **I'll do it later**.

![Prompt to build profile for additional services](screenshots/07-build-profile-prompt.png)

## 10. Land on the Support Portal home

After registration, you're taken to the Broadcom Support Portal home page.

![Support Portal home page](screenshots/08-support-portal-home.jpg)

## 11. Sign in with your new account

Click **Login**, enter your username (the email you registered with), and click **Next**.

![Sign-in username step](screenshots/09-signin-username.png)

## 12. Enter your password

Enter your password and click **Sign In**.

![Sign-in password step](screenshots/10-signin-password.png)

## 13. Reach your Support Portal dashboard

You're now signed in and redirected to the Support Portal dashboard.

![Signed-in Support Portal dashboard](screenshots/11-support-portal-dashboard.jpg)

## 14. Accept the cookie banner

Handle the cookie consent banner (Allow All / Required Only / Cookies Settings) as needed.

![Cookie consent banner](screenshots/12-support-portal-cookies.jpg)

## 15. Find your dashboard via search

The portal doesn't redirect you straight to your dashboard after login. Use the search icon in the top navigation, type **"dashboard"** and press Enter. This opens the AI-powered search results, and the left-hand menu (with **My Dashboard**, **My Entitlements**, **My Downloads**, etc.) becomes available.

![Search results for "dashboard" showing the portal navigation menu](screenshots/13-search-dashboard.png)

## 16. Verify account access via My Downloads

From the left-hand menu, open **My Downloads** to confirm the account is active.

![My Downloads page](screenshots/14-my-downloads-page.png)

## 17. Browse free downloads

Click the **Free Software Downloads** link to see products available without an entitlement, e.g. VMware Workstation Pro.

![List of free VMware downloads](screenshots/15-free-downloads-list.png)

## 18. Pick a product release

Select a product (e.g. VMware Workstation Pro) and choose a release/version.

![VMware Workstation Pro release list](screenshots/16-vmware-workstation-releases.png)

## 19. Review the available files

The product's file list shows release date, checksums (SHA2/MD5), and build number.

![Product files list with checksums](screenshots/17-product-files-list.png)

## 20. Review license terms (optional)

Terms and Conditions link opens Broadcom's Software Agreements and Resources page.

![License and Service Terms page](screenshots/18-license-terms-tab.png)

## 21. Accept the Terms and Conditions

Check **I agree to the Terms and Conditions** before downloading.

![Accepting terms and conditions checkbox](screenshots/19-accept-terms-conditions.png)

## 22. Confirm the screening/verification prompt

Clicking download triggers an additional verification prompt. Click **Yes** to proceed.

![Additional verification required prompt](screenshots/20-screening-verification-prompt.png)

## 23. Fill in the Trade Compliance form

Broadcom requires export-compliance details (name, company, address, country) before releasing the download. Select **I Agree** and click **Submit**.

![Trade Compliance and Download Conditions form](screenshots/21-trade-compliance-form.png)

## 24. Download the file

Back on the product files page, the download icon becomes available.

![Download ready on product files page](screenshots/22-download-ready.png)

## 25. Confirm the file was downloaded

The downloaded installer appears in the local file system, confirming the account works end-to-end.

![Downloaded installer in File Explorer](screenshots/23-downloaded-file-explorer.png)

## 26. Alternative download link and file integrity verification

If the official download link is unavailable or the download fails, you can download the same file from the **SourceForge** mirror.

> **Important:** After downloading the file, it is strongly recommended to verify its integrity before using it.

For example, if you downloaded **VMware-Workstation-Full-26H1-25388281.exe**, verify that its **SHA-256** checksum matches the value published on the official **Broadcom** website:

```text
a0ef9087607d9cad20b08139e73e41242e044ad5bd8cee141d3bad314586737f
```

If both values match exactly, the file has been downloaded correctly and has not been altered.

### 26.1 On Linux

Open a terminal and run the following command:

```bash
sha256sum VMware-Workstation-Full-26H1-25388281.exe
```

Expected output:

```text
a0ef9087607d9cad20b08139e73e41242e044ad5bd8cee141d3bad314586737f  VMware-Workstation-Full-26H1-25388281.exe
```

### 26.2 On Windows (PowerShell)

Open **PowerShell** and run the following command:

```powershell
Get-FileHash ".\VMware-Workstation-Full-26H1-25388281.exe" -Algorithm SHA256
```

Expected output:

```text
Algorithm : SHA256
Hash      : A0EF9087607D9CAD20B08139E73E41242E044AD5BD8CEE141D3BAD314586737F
Path      : C:\Path\To\VMware-Workstation-Full-26H1-25388281.exe
```

The value displayed in the **Hash** field must exactly match the SHA-256 checksum published on the official **Broadcom** website.

> **Note:** If the checksums do not match, **do not run or install the file**. Delete it and download it again from a trusted source.

---

## 27. Summary

1. Register at `profile.broadcom.com/web/registration` with a personal email.
2. Verify the email via the 6-digit code.
3. Fill in personal details and create the account.
4. Sign in at `access.broadcom.com`.
5. Verify access by browsing **My Downloads** and downloading a free product, accepting the license terms and trade compliance form along the way.

## 28. References

- [Broadcom Support Portal](https://support.broadcom.com/)
- [VMware Workstation Pro — Broadcom product page](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
