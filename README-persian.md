[فارسی](README-persian.md) | [English](README.md)

# متریک‌های کاستوم: Node Exporter Textfile و JSON Exporter

خیلی وقت‌ها پیش میاد که می‌خوایم یه سری دیتای خاص و کاستوم رو مانیتور کنیم که ابزارهای استانداردی مثل Node Exporter یا cAdvisor به صورت پیش‌فرض اونا رو بهمون نمیدن. این دیتا می‌تونه خروجی یک اسکریپت ساده، یک لاجیک بیزینسی، یا یک فیلد خاص از یک API تو بستر وب باشه.

در این بخش، ما دو تا روش به شدت منعطف و کاربردی رو یاد می‌گیریم که باهاشون می‌تونیم هر دیتای دلخواهی رو وارد پرومتئوس کنیم:
۱. **ماژول Textfile در Node Exporter**: برای اکسپورت کردن دیتای اسکریپت‌های کاستوم.
۲. **ابزار JSON Exporter**: برای استخراج دیتا از هر API که خروجی JSON میده.

---

## ۱. ماژول Textfile در Node Exporter

نود اکسپورتر (Node Exporter) یه قابلیت داخلی و فوق‌العاده داره به اسم **textfile collector**. کار این ماژول خیلی ساده‌ست: توی یه پوشه مشخص می‌گرده، هر فایلی که پسوندش `.prom` باشه رو می‌خونه، و محتوای اونو به عنوان متریک پرومتئوس اکسپورت می‌کنه!

این روش برای گرفتن متریک از Bash اسکریپت‌ها یا کرون‌جاب‌ها بی‌نظیره (مثلاً بررسی اینکه آیا سرور نیاز به ریبوت داره، شمردن تعداد یوزرهای آنلاین، یا چک کردن حجم پوشه‌ی بک‌آپ‌ها).

### مرحله اول: فعال‌سازی Textfile Collector

برای استفاده از این قابلیت، باید موقع اجرای Node Exporter یک فلگ (Flag) به اسم `--collector.textfile.directory` بهش پاس بدیم.

اگر از **فایل باینری / سرویس systemd** استفاده می‌کنید:
فایل `/etc/systemd/system/node_exporter.service` رو باز کنید و خط `ExecStart` رو به این شکل تغییر بدید:

```ini
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

حالا پوشه رو بسازید و سرویس رو ری‌استارت کنید:

```bash
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

اگر از **Docker Compose** استفاده می‌کنید:
باید به کامند اجرای سرویس این فلگ رو اضافه کنید و پوشه‌ی مورد نظر رو بهش متصل (Mount) کنید:

```diff
    command:
+      - '--collector.textfile.directory=/textfile_metrics'
    volumes:
+      - '/path/to/host/metrics:/textfile_metrics:ro'
```

### مرحله دوم: نوشتن اسکریپت (مثال: تعداد یوزرهای آنلاین)

بیاید یه اسکریپت ساده بنویسیم که تعداد افرادی که الان به سرور وصل هستند رو بشماره و توی یک فایل `.prom` ذخیره کنه.

فایل `/usr/local/bin/metrics_users.sh` رو بسازید:

```bash
#!/usr/bin/env bash

DIR="/var/lib/node_exporter/textfile_collector"
ACTIVE_USERS=$(who | wc -l)
TMP_FILE="$DIR/active_users.prom.$$"

echo "# HELP custom_active_users Number of currently logged in users." > $TMP_FILE
echo "# TYPE custom_active_users gauge" >> $TMP_FILE
echo "custom_active_users $ACTIVE_USERS" >> $TMP_FILE
mv $TMP_FILE $DIR/active_users.prom
```

به اسکریپت دسترسی اجرا بدید:

```bash
sudo chmod +x /usr/local/bin/metrics_users.sh
```

### مرحله سوم: اتوماتیک کردن با Cron

حالا می‌تونید یک کرون‌جاب (Cron Job) تنظیم کنید تا این اسکریپت مثلاً هر دقیقه اجرا بشه:

```bash
crontab -e
```

و این خط رو بهش اضافه کنید:

```bash
* * * * * /usr/local/bin/metrics_users.sh
```

تمام! حالا کافیه به پنل **Prometheus** (یا Grafana Explore) برید و عبارت `custom_active_users` رو سرچ کنید تا دیتای لایو رو ببینید!

---

## ۲. استفاده از JSON Exporter

حالا فرض کنید دیتایی که می‌خواید مانیتور کنید روی وب و داخل یک خروجی JSON قرار داره (مثلاً قیمت بیت‌کوین، وضعیت آب‌وهوا، یا استاتوس یک API خارجی). شما می‌تونید از **JSON Exporter** استفاده کنید تا اون URL رو بخونه، فیلد خاصش رو استخراج کنه و به پرومتئوس تحویل بده.

### مرحله اول: ساخت فایل کانفیگ

فرض کنید می‌خوایم **تعداد دانلودهای (Pulls) یک ایمیج داکر** (مثلاً ایمیج رسمی اوبونتو) رو از API خود Docker Hub مانیتور کنیم. این یک سناریوی بسیار کاربردی توی تیم‌های DevOps هست تا بفهمن ایمیج‌های شرکت چقدر استفاده میشه.
یک فایل به اسم `json_exporter_config.yml` بسازید:

```yaml
modules:
  default:
    metrics:
      - name: dockerhub_pull_count
        # مسیر فیلد مورد نظر در خروجی جیسون (JSONPath)
        path: "{$.pull_count}"
        help: "Total pull count of the Docker Hub repository"
```

### مرحله دوم: اجرای JSON Exporter

شما می‌تونید این اکسپورتر رو با داکر (که خیلی راحت‌تره و پیشنهاد میشه) یا به صورت فایل باینری روی ماشین اجرا کنید.

#### روش الف: نصب فایل باینری (سرویس systemd)

می‌تونید فایل باینری رو مستقیماً دانلود و روی سرور اجرا کنید:

```bash
VERSION=$(curl -s https://api.github.com/repos/prometheus-community/json_exporter/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget -O json_exporter.tar.gz https://github.com/prometheus-community/json_exporter/releases/download/${VERSION}/json_exporter-${VERSION#v}.linux-amd64.tar.gz
tar -xvf json_exporter.tar.gz
sudo mv json_exporter-*/json_exporter /usr/local/bin/
rm -rf json_exporter*
```

برای امنیت بیشتر، یک یوزر سیستمی مجزا و محدود برای اجرای این اکسپورتر بسازید:

```bash
sudo useradd -M -r -s /bin/false json_exporter
```

حالا فایل کانفیگ رو به یک مسیر سیستمی منتقل کنید و دسترسی‌های لازم رو به یوزر بدید:

```bash
sudo mkdir -p /etc/json_exporter
sudo cp json_exporter_config.yml /etc/json_exporter/config.yml
sudo chown -R json_exporter:json_exporter /etc/json_exporter
```

سپس یک سرویس لینوکسی با ساختن فایل `/etc/systemd/system/json_exporter.service` ایجاد کنید:

```ini
[Unit]
Description=JSON Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=json_exporter
Group=json_exporter
ExecStart=/usr/local/bin/json_exporter --config.file=/etc/json_exporter/config.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

در نهایت سرویس رو فعال و استارت کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable json_exporter
sudo systemctl start json_exporter
```

#### روش ب: با استفاده از Docker Compose (پیشنهادی)

اگه ترجیح میدید از داکر استفاده کنید، یک فایل `docker-compose.yml` بسازید:

```yaml
services:
  json_exporter:
    image: prometheuscommunity/json-exporter:latest
    container_name: json_exporter
    restart: unless-stopped
    ports:
      - "7979:7979"
    volumes:
      - ./json_exporter_config.yml:/config.yml:ro
```

و اون رو با `docker compose up -d` اجرا کنید.

### مرحله سوم: تنظیمات پرومتئوس

حالا باید به پرومتئوس بگیم که برای اسکریپ کردن اون API خارجی، از JSON Exporter استفاده کنه. فایل `prometheus.yml` رو باز کنید و این Job رو بهش اضافه کنید:

```yaml
scrape_configs:
  - job_name: 'json_dockerhub'
    metrics_path: /probe
    static_configs:
      - targets:
        - https://hub.docker.com/v2/repositories/library/ubuntu/ # آدرس API مقصد که می‌خوایم دیتای جیسون رو ازش بخونیم
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: {JSONـExporter_IP}:7979 # آی‌پی سروری که JSON Exporter روش نصبه رو به جای {JSONـExporter_IP} قرار بدید
```

در نهایت پرومتئوس رو ری‌استارت کنید.

### مرحله چهارم: مشاهده دیتا!

حالا وارد بخش **Grafana -> Explore** یا پنل پرومتئوس بشید.
عبارت `dockerhub_pull_count` رو سرچ کنید. حالا آمار دقیق دانلود اون ایمیج داکر رو مستقیماً توی گرافانا می‌بینید!

> ### پیشنهاد دوستانه 💡
>
> انعطاف‌پذیری این اکسپورترها واقعاً بی‌نهایته! پیشنهاد می‌کنم توی محیط Grafana Explore کمی بگردید و سعی کنید با این دیتای کاستوم و جدید، داشبوردهای جذاب خودتون رو بسازید! :)
