# Setup Guide — Fonepay QR Data Configuration

The order page needs **one piece of configuration** before it can generate QR codes with the exact amount embedded: your restaurant's Fonepay static QR data string. This guide walks you through getting it and pasting it into `index.html`.

This takes about 5 minutes.

---

## Step 1: Get the Fonepay QR data string

You need the **text content** of your printed Fonepay merchant QR code (the standee at the counter, or the QR shown in your Fonepay Business app). The text is a long string that starts with `00020101` and ends with a 4-character hex CRC.

### Method A: Decode the printed QR code (fastest)

1. Take a clear, well-lit photo of your printed Fonepay merchant QR. Make sure the entire QR is in frame and not blurry.
2. Open any QR code decoder website on your phone or computer. Recommended:
   - **`https://zxing.org/w/decode.jspx`** (works on phone and desktop, no signup, no ads, by Google's ZXing team)
   - Or `https://qrdecoderonline.com`
3. Upload the photo of your QR.
4. The decoder shows the decoded text. It looks something like this:
   ```
   00020101021129...long string of numbers and letters...6304ABCD
   ```
5. **Select the entire string and copy it.** Make sure you copy:
   - From the very first character (`0` of `00020101`)
   - To the very last character (the 4th hex character at the end, after `6304`)
   - No spaces, no line breaks, no extra characters before or after

### Method B: From the Fonepay Business app

1. Open the Fonepay Business app on the owner's phone.
2. Find the static QR display (usually under "My QR" or similar).
3. Take a screenshot.
4. Follow Method A using the screenshot.

### Method C: Ask your bank

If you can't decode the QR yourself, contact Kantipath Branch (the bank you registered with). Tell them you need the **raw QR data string** for the merchant terminal `2222030007913275`. They can email it to you.

---

## Step 2: Paste the QR data into index.html

1. Open `index.html` in any text editor (Notepad, VS Code, or directly on GitHub).
2. Find this line (around line 760, near the top of the `<script>` section):
   ```javascript
   const FONEPAY_QR_DATA = ''; // <-- PASTE YOUR FONEPAY QR DATA HERE
   ```
3. Paste your decoded QR string **between the empty quotes**:
   ```javascript
   const FONEPAY_QR_DATA = '00020101021129...your full string...6304ABCD';
   ```
4. Save the file.

**Common mistakes to avoid:**
- Don't paste spaces or line breaks inside the string
- Don't paste the URL `https://rojinai.github.io/restaurant-order` — that's the page URL, not the QR data
- Don't paste a screenshot/image — the variable expects the **text** that the QR encodes, not the QR itself
- Don't remove the single quotes around the string

---

## Step 3: Upload to GitHub

If editing directly on GitHub:
1. Go to `https://github.com/rojinai/restaurant-order/blob/main/index.html`
2. Click the pencil icon (Edit this file)
3. Paste the new content
4. Scroll to the bottom, click "Commit changes"

If editing locally:
1. Save the file as `index.html`
2. Go to `https://github.com/rojinai/restaurant-order`
3. Click "Add file" → "Upload files"
4. Drag and drop `index.html`
5. Click "Commit changes"

GitHub Pages will automatically rebuild the site within 1-2 minutes.

---

## Step 4: Test before going live

This is critical. Do **not** skip this.

### Test 1: Single item, single wallet
1. On your phone, open the page: `https://rojinai.github.io/restaurant-order`
2. Tap `+` next to "Veg Khana" once. Total should show "Rs. 150" at the bottom.
3. Tap "Show Payment QR →"
4. The payment screen should appear with:
   - Large red "Rs. 150" at the top
   - A QR code in the middle
   - Status text: either "✓ Amount embedded — scan and confirm" (good) or "⚠ Enter amount manually after scanning" (also OK)
   - Your Order ID and Time visible
   - Order summary showing "1 × Veg Khana — Rs. 150"
5. Open your eSewa or Khalti app on another phone and scan the QR.
6. Confirm the wallet shows:
   - Merchant: "Nanu Maya Tamang" (or similar)
   - Amount: pre-filled with `150.00` (if the dynamic QR worked)
   - OR amount blank/zero (if the wallet ignored the dynamic field — you'll enter 150 manually)
7. **Cancel** the payment. Do not actually send money during testing.

### Test 2: Multiple items
1. Reload the page.
2. Tap `+` to get: 2 veg, 1 chicken, 1 mutton. Total = 2(150) + 220 + 290 = **Rs. 810**.
3. Tap "Show Payment QR →"
4. Verify the total amount on screen says "Rs. 810".
5. Scan with a wallet app and verify it shows "810.00" or lets you enter it manually.
6. Cancel.

### Test 3: All four wallet apps your customers use
Repeat Test 1 with each common payment app to make sure the dynamic QR works (or that the fallback is acceptable):
- eSewa
- Khalti
- IME Pay
- ConnectIPS
- Your bank's mobile banking app (Nepal SBI, Nabil, NIC ASIA, NMB, etc.)

**Note** which apps auto-fill the amount and which don't. If most do, you're in great shape — the few that don't will simply show the amount on screen for manual entry, which is the same UX as before.

### Test 4: Real small payment
After tests 1-3 pass, do **one real payment of Rs. 150** to yourself or a trusted person. Confirm:
1. The Fonepay Business app notification arrives within 60 seconds
2. The amount in the notification matches Rs. 150
3. The money is in your bank account

If all four tests pass, you are ready to go live.

---

## Step 5: Print the new counter QR

The counter QR (the printed one customers scan to open this page) doesn't need to change — it still points to `https://rojinai.github.io/restaurant-order`. But if you want to print a new one with better quality:

1. Go to `https://qr-code-generator.com` (or any QR generator)
2. Paste: `https://rojinai.github.io/restaurant-order`
3. Choose "URL" type
4. Generate and download as PNG
5. Print on **laminated paper, at least A5 size**, with the restaurant name and a small logo overlaid as a watermark (this makes sticker fraud visible)
6. Place it at the counter where customers and staff can see it

---

## What to do if something goes wrong

**Page shows "Owner: Fonepay QR not configured":**
You missed Step 2 or pasted incorrectly. Open the source, find `FONEPAY_QR_DATA`, confirm there's a real string between the quotes (not empty, not the placeholder text).

**Page shows "QR rendering failed":**
The string you pasted is probably not a valid EMVCo QR. Re-decode your printed QR (Step 1) and paste again. Make sure you got the FULL string, no truncation.

**Wallet app doesn't show the amount:**
That wallet doesn't read the EMVCo dynamic amount tag for unofficial dynamic QRs. The fallback (showing amount on screen for manual entry) works fine. This is expected for some wallets and is not a bug.

**Wallet app shows wrong merchant name:**
The QR data string you pasted is from a different merchant's QR. Double-check you decoded **your** restaurant's QR, not someone else's.

---

## Going forward

- Keep a backup copy of `index.html` on a USB drive in case GitHub is ever unreachable
- Enable 2FA on the `rojinai` GitHub account (Settings → Password and authentication)
- Read `SECURITY.md` for operational practices around payment verification
- After 1-2 weeks of running this system, review with staff what worked and what didn't, then iterate

That's it. The page is live and serving customers.
