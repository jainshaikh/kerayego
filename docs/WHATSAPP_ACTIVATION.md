# WhatsApp codes — turning them back on

**Status (2026-11-09): WhatsApp verification codes are OFF.** Everything is set up, and the code path was
tested end to end on production, but codes stay off until Meta finishes **business verification**. Until
then people sign in and verify with the testing code (`TEST_ACCOUNT_CODE`), which the owner manages.

When the owner says business verification is done, follow the steps below in order. Read this together
with `kerayego-backend/CLAUDE.md` → "WhatsApp", and use [`API_CONTRACTS.md`](API_CONTRACTS.md) for the
code endpoints.

## What is set up today

| Item | Value |
|---|---|
| Meta app | `1119337397485446` ("KerayeGo"), linked to the WABA (`subscribed_apps`) |
| WhatsApp Business Account (WABA) | `1086508217073067` |
| Sending number | **+92 310 3810011**, Phone number ID `1410186272171902`, CONNECTED, quality GREEN, tier 250 |
| Webhook | `https://kerayego-backend-155868589460.us-east4.run.app/v1/webhooks/whatsapp`. Fields subscribed: `messages`, `message_template_status_update`, `message_template_quality_update` |
| Payment method | Added |
| Templates | 13 `kg_*` UTILITY templates exist (`kg_booking_confirmed` APPROVED, 12 PENDING on 2026-11-09). **`kerayego_verify_code` (the code template) does not exist.** Meta refused it with "This WhatsApp Business account does not have permission to create message template", which needs business verification. |
| Old Meta test number | Phone number ID `1440981715765133`, WABA `1133285459216318`. **No longer used.** Production pointed at it until 2026-11-09. |

**Cloud Run (`kerayego-backend`, region `us-east4`) WhatsApp settings**

| Variable | Value |
|---|---|
| `META_WHATSAPP_PHONE_NUMBER_ID` | `1410186272171902` |
| `META_WHATSAPP_BUSINESS_ACCOUNT_ID` | `1086508217073067` |
| `META_WHATSAPP_BUSINESS_NUMBER` | `923103810011` |
| `META_WHATSAPP_API_VERSION` | `v25.0` |
| `META_WHATSAPP_OTP_TEMPLATE` | `kerayego_verify_code` |
| `META_WHATSAPP_ACCESS_TOKEN`, `META_APP_SECRET`, `META_WHATSAPP_WEBHOOK_VERIFY_TOKEN`, `TEST_ACCOUNT_CODE` | set (secrets) |
| `WHATSAPP_CODES_ENABLED` | **unset = off** |
| `PHONE_VERIFICATION_REQUIRED` | `false` |
| `WHATSAPP_NOTIFICATIONS_ENABLED` | `false` |

Local `kerayego-backend/.env` has the same ids and `WHATSAPP_CODES_ENABLED=false`.

## How the switch works

`WHATSAPP_CODES_ENABLED` is read by `config/whatsapp.config.ts` as `whatsapp.codesEnabled`.
`WhatsAppService.canSendCodes()` returns true only when WhatsApp is configured **and** the switch is
`'true'`.

**While it's off:**
- `POST /auth/phone/request` and `POST /users/me/phone/send-code` send nothing and include no
  `messageUsUrl`.
- They answer `codeSent: false` with the message "Enter your 6-digit code.". This is 200 while the testing
  code is on. Without the testing code, production answers 503 `WHATSAPP_UNAVAILABLE`.
- The webhook ignores "code" messages: `PhoneCodeByMessageService.reply` returns `'off'` and sends nothing.
- Mobile and the Partner app show their normal code step without the WhatsApp button.
- Outside production, the code is printed in the backend console.

**When it's on, there are two ways to deliver a code:**
1. **Code template** (`kerayego_verify_code`, AUTHENTICATION): the backend sends the code by itself. This
   works only once the template exists and is approved, which needs business verification.
2. **"Message us for your code"** (needs `META_WHATSAPP_BUSINESS_NUMBER`): if the template send fails, the
   app shows "Get the code on WhatsApp". The person sends the pre-filled "Send my KerayeGo code" and the
   webhook replies with the code as plain text. This was tested on production on 2026-11-09 from
   +923313693668 and the reply was delivered. It stays as the backup after activation.

**The 24-hour rule:** a business can start a chat only with an approved template. Free text is allowed
only within 24 h after the person messages first. That's why path 2 needs one message from the user each
time, and why path 1 needs the template.

## Steps to turn codes on (after business verification)

1. **Owner confirms verification.** Business Settings → Security Center shows the business as Verified.
2. **Claude creates the code template.** In `kerayego-backend`, run `npm run whatsapp:templates`. It uses
   the local `.env`, which points at the real WABA, and creates only the missing templates. The output
   must show `kerayego_verify_code` created. If it again says "does not have permission", verification
   isn't finished yet, so stop and tell the owner.
3. **Wait for the template to be APPROVED.** Check with:
   ```bash
   cd kerayego-backend && T=$(grep -E "^META_WHATSAPP_ACCESS_TOKEN=" .env | cut -d= -f2- | tr -d '"\r')
   curl -s "https://graph.facebook.com/v25.0/1086508217073067/message_templates?fields=name,status&limit=50" -H "Authorization: Bearer $T"
   ```
4. **Owner turns the switch on in Cloud Run**, either in the console (Edit & deploy new revision →
   Variables) or with:
   ```bash
   gcloud run services update kerayego-backend --region us-east4 --update-env-vars WHATSAPP_CODES_ENABLED=true
   ```
   Claude can't run this: auto mode blocks production changes. Give the owner the command.
5. **Test on production.** Ask the owner which number to use; +923313693668 was the test phone.
   ```bash
   curl -s -X POST https://kerayego-backend-155868589460.us-east4.run.app/v1/auth/phone/request \
     -H "Content-Type: application/json" -d '{"phone":"+923313693668"}'
   ```
   Expect `codeSent: true`, with the code arriving on WhatsApp by itself. `codeSent: false` with a
   `messageUsUrl` means the template send failed. Read the reason in the logs:
   ```bash
   gcloud logging read 'resource.type=cloud_run_revision AND resource.labels.service_name=kerayego-backend AND textPayload:"WhatsApp"' --limit=20 --format="value(timestamp,textPayload)"
   ```
   Then the owner signs in from the mobile app, the Partner app and **web**. Web has no WhatsApp button:
   it depends on the template, so this is the first time web gets real codes.
6. **Fix shared phone numbers before relying on phone sign-in.** Phone sign-in finds the account with
   `findFirst({ phone })`, so a number on several accounts signs in to an arbitrary one. The unique rule
   `users_phone_live_key` was skipped by migration `20261101120000_email_code_login_unique_phone` because
   duplicates existed. On 2026-11-09 three numbers were shared:

   | Number | Accounts |
   |---|---|
   | `+923313693668` | 7 accounts (the owner's test number; `jainshaikh@gmail.com` is the only verified one) |
   | `+923128933120` | 4 accounts (`haseeb…`) |
   | `+923163354063` | 2 accounts (`syedasadali@yopmail.com`, `sa4796728@gmail.com`) |

   The owner decides which account keeps each number. Remove it from the others with the admin endpoint
   `POST /v1/admin/users/:id/release-phone`, then create the index by hand (the SQL is in that migration's
   header). Re-check first with the duplicate query in the same header.
7. **Owner removes `TEST_ACCOUNT_CODE`** from Cloud Run once real codes work for everyone. The seeded test
   accounts (`+923000000101`…`106`) then need real WhatsApp numbers to sign in by phone.
8. **Owner decides when to set `PHONE_VERIFICATION_REQUIRED=true`.** After that, every non-admin account
   must verify its number.
9. **Notifications are separate.** When the 12 `kg_*` templates are APPROVED, set
   `WHATSAPP_NOTIFICATIONS_ENABLED=true`. This doesn't need business verification, but before
   verification Meta's tier limits the number of people messaged per day.
10. **Update the docs.** Change the status line at the top of this file, the `WHATSAPP_CODES_ENABLED` rows in
    [`ENVIRONMENT_GUIDE.md`](ENVIRONMENT_GUIDE.md) and [`API_CONTRACTS.md`](API_CONTRACTS.md), the
    WhatsApp checklist in `PROVIDER_APP_PLAN.md`, and `kerayego-backend/CLAUDE.md`.

## Turning it off again

Set `WHATSAPP_CODES_ENABLED` to anything but `true`, or remove it, and make sure `TEST_ACCOUNT_CODE` is
set. Otherwise production sign-in answers 503.
