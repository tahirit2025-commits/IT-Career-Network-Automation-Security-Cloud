Windows Deployment Services (WDS) 
— راهنمای جامع برای Windows Server 2022
مقدمه

Windows Deployment Services (WDS) یک سرویس مایکروسافتی است که امکان نصب سیستم‌عامل ویندوز را از طریق شبکه (PXE) فراهم می‌کند. در محیط‌های دانشگاهی، لابراتوارها و سازمان‌ها که تعداد زیادی دستگاه باید نصب یا بازنشانی شوند، WDS باعث صرفه‌جویی در زمان، ایجاد یک فرآیند استاندارد و کاهش خطاهای دستی می‌شود. این راهنما مخصوص Windows Server 2022 نوشته شده و گام‌به‌گام همراه با توضیحات فنی، دستورات PowerShell و نکات عیب‌یابی ارائه می‌دهد.


---

فهرست محتوا

1. اهداف و کاربردها


2. پیش‌نیازها و توپولوژی نمونه


3. نصب نقش WDS (GUI و PowerShell)


4. پیکربندی اولیه WDS


5. افزودن Boot و Install Image


6. ساخت و Capture یک Image سفارشی (Sysprep + Capture)


7. راه‌اندازی کلاینت‌ها با PXE (BIOS/UEFI)


8. گزینه‌های DHCP و تعامل با سرور DHCP جداگانه


9. مدیریت Image با DISM و PowerShell


10. بهینه‌سازی: Multicast، Unattended Answer Files، Driver Injection


11. امنیت، مجوزها و فایروال


12. پشتیبان‌گیری و بازیابی


13. خطاهای رایج و روش‌های حل آنها


14. لابراتوار پیشنهادی و پروژه دانشگاهی


15. منابع و ادامهٔ کار (پیشنهاد مطالعه/تمرین)




---

1 — اهداف و کاربردها

اتوماسیون نصب ویندوز روی چندین دستگاه.

ایجاد Imageهای استاندارد سازمانی شامل نرم‌افزارها، تنظیمات امنیتی و پیکربندی‌ها.

کاهش زمان نصب و اشتباهات انسان‌محور.

راه‌اندازی تصویری برای بازیابی سریع سیستم‌ها.



---

2 — پیش‌نیازها و توپولوژی نمونه

اجزای لازم:

Windows Server 2022 (Domain Controller و/یا عضو دامنه)

نقش WDS نصب‌شده روی سرور اختصاصی (ترجیحاً عضو دامنه)

سرور DHCP (می‌تواند هم‌سرور یا جداگانه باشد)

سرور DNS (برای نام‌گذاری شبکه)

کلاینت‌های هدف که PXE Boot را پشتیبانی کنند (UEFI و/یا Legacy BIOS)

فایل ISO ویندوز (برای بدست آوردن boot.wim و install.wim)

فضای دیسک مناسب برای نگهداری RemoteInstall (پیشنهاد حداقل 50–100 گیگابایت بسته به تعداد imageها)


توپولوژی نمونه ساده:

[Internet/Local Repos]
        |
     [DNS/DHCP]
        |
   [WDS Server] --- (PXE) ---> [Clients]


---

3 — نصب نقش WDS

روش گرافیکی (Server Manager)

1. Server Manager → Add Roles and Features


2. Role-based or feature-based installation → انتخاب سرور


3. انتخاب Windows Deployment Services


4. در مرحلهٔ Features معمولاً نیازی به انتخاب چیز دیگر نیست.


5. نصب و پس از اتمام گزینهٔ Configure Server را اجرا کن.



نصب با PowerShell (سریع و قابل تکرار)

Install-WindowsFeature -Name WDS, WDS-AdminPack -IncludeManagementTools

> توضیح: WDS-AdminPack ابزار مدیریت را نصب می‌کند. اجرای PowerShell برای خودکارسازی در سناریوهای لابراتوار یا چند سروری مفید است.




---

4 — پیکربندی اولیه WDS (Windows Server 2022)

پس از نصب باید سرور را پیکربندی کنید.

با رابط گرافیکی

1. از Server Manager یا منوی Tools وارد Windows Deployment Services شوید.


2. روی نام سرور راست‌کلیک → Configure Server.


3. انتخاب گزینه‌های زیر:

Integration with Active Directory (اگر عضو دامنه است) → توصیه می‌شود.

انتخاب مسیر ذخیره‌سازی Image (مثلاً D:\RemoteInstall) — مسیر باید درایو محلی یا shared نباشد (فایل‌ها توسط WDS مدیریت می‌شوند).



4. تنظیمات PXE response:

پاسخ به همهٔ PXE requests یا فقط به کلاینت‌های درون دامنه. (در لابراتوار: پاسخ به همه را انتخاب کن)




با PowerShell

اول، پوشهٔ RemoteInstall را ایجاد کن و سپس سرویس را فعال کن:

New-Item -Path "D:\RemoteInstall" -ItemType Directory
wdsutil /initialize-server /reminst:"D:\RemoteInstall" /architecture:x64

> توجه: wdsutil ابزار خط فرمان مخصوص WDS است و برای برخی عملیات پیکربندی مفید است.




---

5 — افزودن Boot Image و Install Image

برای راه‌اندازی PXE دو نوع Image نیاز داریم: Boot Image (معمولاً WinPE) و Install Image (فایل نصب ویندوز).

گرفتن فایل‌ها از ISO ویندوز

Mount کن ISO ویندوز (ویندوز 10/11 یا سرور).

مسیر /sources/boot.wim (برای Boot Image)

مسیر /sources/install.wim یا /sources/install.esd (برای Install Image)


افزودن با GUI

1. WDS Console → Boot Images → راست‌کلیک → Add Boot Image → انتخاب boot.wim


2. WDS Console → Install Images → ایجاد Image Group → راست‌کلیک → Add Install Image → انتخاب install.wim



افزودن با PowerShell

Import-WdsBootImage -Path "D:\ISOs\Win11\sources\boot.wim" -Name "WinPE x64"
Import-WdsInstallImage -Path "D:\ISOs\Win11\sources\install.wim" -ImageGroup "Windows 11"


---

6 — ساخت Image سفارشی و Capture (Sysprep)

برای داشتن یک Image سازمانی که نرم‌افزارها و تنظیمات موردنظر را دارد، روی یک ماشین مرجع عمل کنید.

گام‌ها به ترتیب:

1. یک ماشین مجازی یا فیزیکی نصب کن و ویندوز را نصب کن.


2. همه نرم‌افزارها، به‌روزرسانی‌ها و تنظیمات امنیتی را اعمال کن.


3. پاکسازی نهایی: پاک‌کردن فایل‌های موقتی، logها و غیره.


4. اجرای Sysprep:



C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown

/generalize : حذف شناسه‌های خاص سخت‌افزار.

/oobe : نمایش صفحهٔ OOBE برای اولین راه‌اندازی.

/shutdown : خاموش شدن پس از sysprep (برای گرفتن تصویر).


5. بوت ماشین مرجع با یک Boot WinPE که در WDS ثبت شده است (PXE boot).


6. در WDS Console یا از طریق Capture Image یک Image از ماشین مرجع بگیر:

WDS → Boot Images → انتخاب Boot Image → راست‌کلیک → Create Capture Image

سپس از طریق PXE با ابزار Capture image روی WinPE، تصویر را به سرور ارسال کن.
یا از PowerShell:




# از کامندلت‌های wdsutil یا ابزارهای دیگر برای Capture استفاده می‌شود؛ مراحل GUI و WinPE متداول‌تر هستند.

> نکته: Capture Image معمولاً یک Boot Image است که ابزار capture را دارد.




---

7 — Boot کلاینت‌ها با PXE (BIOS و UEFI)

کلاینت‌ها هنگام روشن شدن باید از شبکه بوت شوند.

مواردی که باید چک شوند:

تنظیمات BIOS/UEFI: Network Boot یا PXE فعال باشد.

ترتیب بوت: Network boot بالاتر از هارد قرار گیرد (یا به طور موقت از منوی Boot انتخاب شود).

برای UEFI مطمئن شو که Boot Image پشتیبانی UEFI اضافه شده است (بعضی WIMها هر دو را پشتیبانی می‌کنند، اما گاهی لازم است نسخهٔ x64/UEFI جداگانه).


تفاوت‌های مهم:

Legacy BIOS از فایل pxeboot.com/wdsnbp.com استفاده می‌کند.

UEFI از wdsmgfw.efi یا مشابه آن استفاده می‌کند.
WDS به طور خودکار از Imageهای مناسب (UEFI/BIOS) استفاده می‌کند اگر Imageها و کلاینت‌ها سازگار باشند.



---

8 — تعامل WDS با DHCP (گزینه‌ها و بهترین روش‌ها)

اگر DHCP روی همان سرور WDS نصب شده باشد، WDS می‌تواند با DHCP همکاری کند اما باید تنظیمات ویژه‌ای انجام شود.

سناریوها:

1. DHCP و WDS روی یک سرور

در WDS Server: Do not listen on port 67 را تیک بزن (در تنظیمات WDS) یا با wdsutil تنظیم کن تا با DHCP تداخل نکند.



2. DHCP جداگانه

روی سرور DHCP دو Option رایج را تنظیم کن:

Option 66 — Boot Server Host Name (IP یا نام WDS)

Option 67 — Bootfile Name (برای UEFI/BIOS متفاوت است، مثال: boot\x64\wdsnbp.com یا مسیر فایل EFI برای UEFI)


یا از ProxyDHCP استفاده شود: WDS به عنوان ProxyDHCP پاسخ PXE را می‌دهد و DHCP آدرس IP را.




توصیه عملی

در محیط‌های تولیدی بهتر است از DHCP جداگانه استفاده شود و Option 66/67 را فقط در صورت نیاز تنظیم کنید. در لابراتوار می‌توانید همه چیز را روی یک سرور داشته باشید ولی مراقب مسائل پورت و تداخل باش.


---

9 — مدیریت Image با DISM و PowerShell

بررسی محتویات یک WIM

# فهرست ایندکس‌های داخل install.wim
dism /Get-WimInfo /WimFile:C:\path\to\install.wim

افزودن/حذف درایور از WIM

# Mount کردن WIM
dism /Mount-Wim /WimFile:C:\images\install.wim /Index:1 /MountDir:C:\mount

# افزودن درایور
dism /Image:C:\mount /Add-Driver /Driver:C:\drivers\driver.inf

# Unmount و Commit
dism /Unmount-Wim /MountDir:C:\mount /Commit

استفاده از PowerShell (برای WDS)

# فهرست تصاویر موجود روی WDS
Get-WdsInstallImage

# حذف یک تصویر
Remove-WdsInstallImage -ImageGroup "Windows 11" -ImageName "Windows 11 Pro"

# وارد کردن تصویر بوت یا نصب
Import-WdsInstallImage -Path "D:\ISOs\Win11\sources\install.wim" -ImageGroup "Windows 11"


---

10 — بهینه‌سازی و امکانات پیشرفته

Multicast

برای نصب همزمان روی چند کلاینت، Multicast را فعال کن تا یک تصویر به‌صورت یکبار روی شبکه فرستاده شود و کلاینت‌ها به آن متصل شوند. این به‌خصوص در محیط‌های بزرگ بسیار کاربردی است.

در WDS می‌توان Sessionهای Multicast تعریف کرد (Unicast, Scheduled, Optimized).


Unattended (Answer Files)

فایل‌های XML (که با Windows System Image Manager ساخته می‌شوند) امکان نصب بدون دخالت کاربر، autopartition و تخصیص تنظیمات را فراهم می‌کنند.

فایل‌های unattended را می‌توان در WDS به Install Imageها متصل کرد یا در فایل Boot Image پیکربندی کرد.


Driver Injection

قبل از دیپلوی، درایورهای سخت‌افزار کلاینت‌ها (مثلاً درایور شبکه برای PXE) را در Boot Image یا Install Image قرار دهید.

از dism یا Add-WindowsDriver استفاده کنید.



---

11 — امنیت، مجوزها و فایروال

WDS معمولاً نیاز به دسترسی نوشتن به پوشهٔ RemoteInstall دارد؛ اطمینان حاصل کنید که فقط مدیران و سرویس‌های مورد اعتماد دسترسی داشته باشند.

اگر WDS عضو دامنه است، از AD-integrated permissions استفاده کنید.

پورت‌های شبکه: برای PXE و WDS به پورت‌های DHCP (67/68)، TFTP (69)، RPC/SMB برای انتقال فایل‌ها و HTTP/HTTPS در حالت‌های جدید نیاز است؛ فایروال ویندوز را مطابق نیاز پیکربندی کن.

برای جلوگیری از نصب غیراصولی، پاسخ PXE را محدود به کلاینت‌های شناخته‌شده یا اعضای دامنه کن.



---

12 — پشتیبان‌گیری و بازیابی

همیشه از پوشهٔ RemoteInstall پشتیبان بگیر (فایل‌های WIM، XMLها و فایل‌های پیکربندی).

دستورالعمل بازگردانی:

1. نصب مجدد نقش WDS.


2. Initialize-server به همان مسیر RemoteInstall (یا ریستور مسیر).


3. ریستور فایل‌های Image و تنظیمات.



همچنین، از AD و DHCP پیکربندی‌ها بکاپ بگیر.



---

13 — خطاهای رایج و رفع آن‌ها

PXE-E53: No boot filename received
دلیل: Option 67 تنظیم نشده یا ProxyDHCP مشکل دارد.
راه‌حل: بررسی DHCP Options و اطمینان از اینکه WDS به درستی پاسخ PXE را ارسال می‌کند.

PXE-E55: ProxyDHCP offers were not received
دلیل: تداخل DHCP/WDS یا مسدود شدن پورت‌ها.
راه‌حل: اگر DHCP جداست، Option 66/67 را تنظیم کن یا در صورت هم‌سروری، WDS را طوری پیکربندی کن که با DHCP تداخل نداشته باشد.

WDSClient: No image found
دلیل: Install Image ثبت نشده یا گروه تصویر خالی است.
راه‌حل: بررسی WDS Console و مطمئن شدن از وجود Boot & Install Images.

Access denied to WDS server
دلیل: مجوزها یا DNS اشتباه.
راه‌حل: بررسی DNS، اطمینان از Join بودن کلاینت به دامنه (در صورت محدودسازی) و تنظیم مجوزها.



---

14 — لابراتوار پیشنهادی (گام‌به‌گام برای تمرین)

Lab A — نصب پایه WDS روی Windows Server 2022

1. فراهم کردن VM: نصب Windows Server 2022، عضو کردن در دامنه یا نگه داشتن به عنوان standalone.


2. نصب نقش WDS (PowerShell یا GUI).


3. پیکربندی اولیه و ایجاد D:\RemoteInstall.


4. افزودن Boot Image (boot.wim) و Install Image (install.wim) از ISO.



Lab B — ساخت یک Image سازمانی

1. ایجاد VM مرجع، نصب ویندوز و نرم‌افزارها.


2. اجرای Sysprep و shutdown.


3. PXE Boot با Boot Image و گرفتن Capture Image از ماشین مرجع.


4. افزودن Image Capture شده به WDS و آزمایش deploy روی یک VM جدید.



Lab C — پیاده‌سازی Unattended و Multicast

1. ساخت فایل unattend.xml با Windows SIM.


2. اتصال فایل Unattend به Install Image در WDS.


3. ایجاد Session Multicast و دیپلوی همزمان به چند کلاینت.




---

15 — نکات عملی و بهترین شیوه‌ها

همیشه یک Image مرجع تمیز و نگهداری‌شده داشته باش؛ مرتبا آن را با آخرین آپدیت‌ها بروزرسانی کن.

از اسکریپت‌های PowerShell برای اتوماسیون وارد کردن Imageها و مدیریت استفاده کن.

فضای دیسک کافی برای RemoteInstall را در نظر داشته باش؛ WIMها می‌توانند بزرگ باشند.

مستندسازی کامل تنظیمات DHCP/Option و فایل‌های Unattend را نگه دار.

تست عملی در محیط لابراتوار قبل از پیاده‌سازی در محیط تولید.



---

پیوست — دستورات و اسکریپت‌های مفید (نمونه)

بررسی وضعیت سرویس WDS

Get-Service -Name WDSServer

لیست کردن Boot/Install Imageها

Get-WdsBootImage
Get-WdsInstallImage

اضافه کردن Install Image (مثال)

Import-WdsInstallImage -Path "D:\ISOs\Win11\sources\install.wim" -ImageGroup "Windows 11"

حذف تصویر

Remove-WdsInstallImage -ImageGroup "Windows 11" -ImageName "Windows 11 Pro"

نمونه: Initialize WDS با wdsutil

wdsutil /initialize-server /reminst:D:\RemoteInstall


---

جمع‌بندی

این سند راهنمایی جامع و کاربردی برای راه‌اندازی و مدیریت Windows Deployment Services روی Windows Server 2022 ارائه داد. با دنبال کردن گام‌ها، می‌توانید یک محیط حرفه‌ای برای دیپلوی سیستم‌عامل‌ها بسازید که مقیاس‌پذیر، قابل اتوماسیون و ایمن باشد.
