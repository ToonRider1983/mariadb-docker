# MariaDB Docker

A Docker Compose project for running MariaDB with phpMyAdmin, a browser-based database administration tool.

โปรเจกต์ Docker Compose สำหรับรัน MariaDB พร้อม phpMyAdmin ซึ่งเป็นเครื่องมือจัดการฐานข้อมูลผ่านเว็บเบราว์เซอร์

- [English](#english)
- [ภาษาไทย](#ภาษาไทย)

## English

### Overview

This repository provides two services on a shared Docker bridge network. MariaDB stores database files in a named volume, while phpMyAdmin connects to MariaDB using its Compose service name.

| Component | Current configuration |
| --- | --- |
| Database | `mariadb:lts`, container name `mariadb` |
| Database access from the host | `localhost:3306` |
| Administration UI | `phpmyadmin:latest`, container name `phpmyadmin` |
| Web interface | http://localhost:8080 |
| Persistent storage | `mariadb_data` mounted at `/var/lib/mysql` |
| Network | `database_network`, bridge driver |
| Restart policy | `unless-stopped` for both services |

The image tags `lts` and `latest` are not pinned to exact versions. Pulling images later can therefore change the versions used by this project.

### Repository structure

```text
mariadb-docker/
├── .env                      # Database initialization settings
├── docker-compose.yaml       # Services, ports, volume, and network
├── phpmyadmin/
│   └── custome.ini           # Additional PHP setting; currently not mounted
└── README.md
```

`phpmyadmin/custome.ini` contains `max_input_time=5000`. The Compose file does not mount this file into phpMyAdmin, so this setting is currently inactive. The filename is spelled `custome.ini` in this repository.

### Prerequisites

- Docker Engine with the Docker Compose plugin, or Docker Desktop with Linux containers enabled.
- A running Docker daemon.
- Available host ports `3306` and `8080`.

Check that Docker and Compose are available:

```sh
docker --version
docker compose version
```

Run the following commands from the repository root, where `docker-compose.yaml` is located.

### Configuration

Edit the existing `.env` file before the first startup. If it is missing, create it with these keys and replace the example values:

```dotenv
MARIADB_ROOT_PASSWORD=replace_with_a_strong_root_password
MARIADB_DATABASE=app_database
MARIADB_USER=app_user
MARIADB_PASSWORD=replace_with_a_strong_user_password
```

| Variable | Purpose |
| --- | --- |
| `MARIADB_ROOT_PASSWORD` | Password for the MariaDB `root` administrator account |
| `MARIADB_DATABASE` | Database created during initial setup |
| `MARIADB_USER` | Application user created during initial setup |
| `MARIADB_PASSWORD` | Password for the application user |

These initialization settings apply when MariaDB starts with an empty data directory. Editing `.env` after the database has been initialized does not automatically change existing users, passwords, or databases. Update an existing database through SQL or phpMyAdmin instead.

The repository currently tracks `.env` in Git. Do not commit real credentials; use local secrets and remove sensitive values from version control before sharing or deploying this project.

### Start the services

```sh
# Check Compose configuration without printing resolved credentials
docker compose config --quiet

# Download images and start in the background
docker compose pull
docker compose up -d

# Check status and database startup logs
docker compose ps
docker compose logs -f mariadb
```

Press `Ctrl+C` to stop following logs; the containers continue running. Initial database setup can take some time. `depends_on` starts MariaDB before phpMyAdmin, but the project has no database health check and does not wait for MariaDB to be ready to accept connections.

### Connect to the database

**phpMyAdmin:** Open http://localhost:8080 and sign in with the `MARIADB_USER` and `MARIADB_PASSWORD` values from `.env`. Use `root` and `MARIADB_ROOT_PASSWORD` only when administrator access is needed. phpMyAdmin is already configured to connect to `mariadb:3306`; no database host entry is needed in the login form.

**Desktop database client or application running on the host:**

| Field | Value |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `3306` |
| Database | Value of `MARIADB_DATABASE` |
| Username | Value of `MARIADB_USER` |
| Password | Value of `MARIADB_PASSWORD` |

**Another container:** Attach it to the same Docker network and use `mariadb` as the hostname with port `3306`. Inside a container, `localhost` refers to that container itself. Compose normally prefixes the actual network and volume names with the project name.

**MariaDB command-line client:**

```sh
docker compose exec mariadb mariadb -u root -p
```

Enter the root password when prompted. Use `exit` to leave the SQL client.

### phpMyAdmin limits

The Compose file configures these phpMyAdmin environment variables:

| Setting | Value | Purpose |
| --- | --- | --- |
| `PMA_HOST` | `mariadb` | Database hostname on the Docker network |
| `PMA_PORT` | `3306` | Internal database port |
| `UPLOAD_LIMIT` | `750M` | Upload size setting |
| `MAX_EXECUTION_TIME` | `5000` | PHP execution time setting, in seconds |
| `MEMORY_LIMIT` | `1000M` | PHP memory limit setting |

These are PHP/phpMyAdmin settings, not Docker resource limits. Large imports may still fail because of other timeouts or available resources. The separate `max_input_time` setting in `custome.ini` is not applied by the current Compose configuration.

After editing Compose environment settings, recreate the affected service:

```sh
docker compose up -d phpmyadmin
```

### Everyday commands

| Action | Command |
| --- | --- |
| Show service status | `docker compose ps` |
| Follow all service logs | `docker compose logs -f` |
| Follow phpMyAdmin logs | `docker compose logs -f phpmyadmin` |
| Stop containers without removing them | `docker compose stop` |
| Start stopped containers | `docker compose start` |
| Restart services | `docker compose restart` |
| Remove containers and the Compose network, keeping database data | `docker compose down` |
| Recreate services after configuration changes | `docker compose up -d` |

### Data persistence and backups

Database files are stored in the `mariadb_data` named volume. They survive container restarts, recreation, and a normal `docker compose down`. The volume is persistent storage, not a backup.

For a manual backup, use phpMyAdmin: select the database, choose **Export**, select SQL format, and download the file. To restore, select the target database and use **Import** with the SQL file. Store backups outside Docker and verify that they can be restored, especially before image upgrades.

To completely reset the project, the following command deletes the database volume and **permanently removes its data**. Back up anything you need first:

```sh
docker compose down --volumes
docker compose up -d
```

The next startup initializes a fresh database using the current `.env` values.

### Troubleshooting and deployment notes

- **Port already in use:** Stop the conflicting service or change the host side of a port mapping, for example `3307:3306` or `8081:80`. Host clients must then use the new host port; phpMyAdmin still connects internally to `mariadb:3306`.
- **Container name already in use:** The fixed names `mariadb` and `phpmyadmin` can conflict with existing containers or a second copy of this project. Change or remove `container_name` when running multiple stacks.
- **Connection refused immediately after startup:** Check `docker compose logs mariadb` and wait for database initialization to finish.
- **Access denied after changing `.env`:** Existing credentials remain in the data volume. Use the existing administrator credentials to update accounts; do not delete the volume unless a full reset is intended.
- **PHP configuration changes have no effect:** `custome.ini` is not mounted. Add an appropriate PHP configuration mount before expecting that file to take effect.
- **Network exposure:** The current port mappings do not restrict bindings to localhost. For local-only access, change them to `127.0.0.1:3306:3306` and `127.0.0.1:8080:80`.
- **Production use:** This repository does not configure HTTPS, automated backups, health checks, or Docker resource limits. Review access controls, secret handling, backup procedures, and image version pinning before deployment. The provided phpMyAdmin URL uses HTTP.

## ภาษาไทย

### ภาพรวม

โปรเจกต์นี้ประกอบด้วยบริการสองตัวที่สื่อสารกันผ่านเครือข่าย Docker แบบ bridge โดย MariaDB เก็บไฟล์ฐานข้อมูลใน named volume และ phpMyAdmin เชื่อมต่อฐานข้อมูลผ่านชื่อบริการใน Compose

| ส่วนประกอบ | การตั้งค่าปัจจุบัน |
| --- | --- |
| ฐานข้อมูล | `mariadb:lts` ชื่อคอนเทนเนอร์ `mariadb` |
| การเชื่อมต่อฐานข้อมูลจากเครื่องโฮสต์ | `localhost:3306` |
| เครื่องมือจัดการฐานข้อมูล | `phpmyadmin:latest` ชื่อคอนเทนเนอร์ `phpmyadmin` |
| หน้าเว็บ | http://localhost:8080 |
| พื้นที่จัดเก็บถาวร | `mariadb_data` เชื่อมเข้ากับ `/var/lib/mysql` |
| เครือข่าย | `database_network` ใช้ไดรเวอร์ `bridge` |
| นโยบายรีสตาร์ต | `unless-stopped` สำหรับทั้งสองบริการ |

แท็กอิมเมจ `lts` และ `latest` ไม่ได้ระบุเวอร์ชันตายตัว การดึงอิมเมจในภายหลังจึงอาจทำให้เวอร์ชันที่ใช้งานเปลี่ยนไป

### โครงสร้างไฟล์

```text
mariadb-docker/
├── .env                      # ค่าที่ใช้เริ่มต้นฐานข้อมูล
├── docker-compose.yaml       # บริการ พอร์ต volume และเครือข่าย
├── phpmyadmin/
│   └── custome.ini           # การตั้งค่า PHP เพิ่มเติมที่ยังไม่ได้ mount
└── README.md
```

ไฟล์ `phpmyadmin/custome.ini` มีค่า `max_input_time=5000` แต่ Compose ยังไม่ได้ mount ไฟล์นี้เข้าไปใน phpMyAdmin จึงยังไม่มีผลต่อการทำงาน โดยชื่อไฟล์ในโปรเจกต์สะกดว่า `custome.ini`

### สิ่งที่ต้องเตรียม

- Docker Engine พร้อมปลั๊กอิน Docker Compose หรือ Docker Desktop ที่เปิดใช้ Linux containers
- Docker daemon ที่กำลังทำงาน
- พอร์ต `3306` และ `8080` บนเครื่องโฮสต์ที่ยังไม่ถูกใช้งาน

ตรวจสอบ Docker และ Compose ด้วยคำสั่ง:

```sh
docker --version
docker compose version
```

ให้รันคำสั่งในเอกสารนี้จากโฟลเดอร์หลักของโปรเจกต์ ซึ่งเป็นตำแหน่งเดียวกับ `docker-compose.yaml`

### การตั้งค่า

แก้ไขไฟล์ `.env` ที่มีอยู่ก่อนเริ่มใช้งานครั้งแรก หากไม่มีไฟล์ ให้สร้างโดยใช้ชื่อตัวแปรดังนี้ และเปลี่ยนค่าตัวอย่างเป็นค่าของคุณ:

```dotenv
MARIADB_ROOT_PASSWORD=replace_with_a_strong_root_password
MARIADB_DATABASE=app_database
MARIADB_USER=app_user
MARIADB_PASSWORD=replace_with_a_strong_user_password
```

| ตัวแปร | ความหมาย |
| --- | --- |
| `MARIADB_ROOT_PASSWORD` | รหัสผ่านของผู้ดูแลฐานข้อมูล `root` |
| `MARIADB_DATABASE` | ชื่อฐานข้อมูลที่จะสร้างในการเริ่มต้นครั้งแรก |
| `MARIADB_USER` | ชื่อผู้ใช้สำหรับแอปพลิเคชันที่จะสร้างในการเริ่มต้นครั้งแรก |
| `MARIADB_PASSWORD` | รหัสผ่านของผู้ใช้สำหรับแอปพลิเคชัน |

ค่าเหล่านี้ใช้สำหรับเริ่มต้น MariaDB เมื่อไดเรกทอรีข้อมูลยังว่างอยู่เท่านั้น การแก้ไข `.env` หลังจากสร้างฐานข้อมูลแล้วจะไม่เปลี่ยนผู้ใช้ รหัสผ่าน หรือฐานข้อมูลเดิมโดยอัตโนมัติ ให้แก้ไขฐานข้อมูลที่มีอยู่ผ่าน SQL หรือ phpMyAdmin

ปัจจุบัน Git ติดตามไฟล์ `.env` ในโปรเจกต์นี้อยู่ อย่า commit รหัสผ่านจริง ควรเก็บข้อมูลลับไว้ในเครื่องและนำค่าที่เป็นความลับออกจากระบบควบคุมเวอร์ชันก่อนแชร์หรือนำโปรเจกต์ไปใช้งาน

### เริ่มใช้งาน

```sh
# ตรวจสอบ Compose โดยไม่แสดงค่ารหัสผ่านที่แทนค่าแล้ว
docker compose config --quiet

# ดาวน์โหลดอิมเมจและเริ่มทำงานเบื้องหลัง
docker compose pull
docker compose up -d

# ตรวจสอบสถานะและติดตาม log ของฐานข้อมูล
docker compose ps
docker compose logs -f mariadb
```

กด `Ctrl+C` เพื่อหยุดติดตาม log โดยคอนเทนเนอร์ยังทำงานต่อ การสร้างฐานข้อมูลครั้งแรกอาจใช้เวลาสักครู่ `depends_on` กำหนดให้เริ่ม MariaDB ก่อน phpMyAdmin แต่โปรเจกต์ยังไม่มี health check และไม่ได้รอจนฐานข้อมูลพร้อมรับการเชื่อมต่อ

### การเชื่อมต่อฐานข้อมูล

**ผ่าน phpMyAdmin:** เปิด http://localhost:8080 แล้วเข้าสู่ระบบด้วยค่า `MARIADB_USER` และ `MARIADB_PASSWORD` จาก `.env` ใช้ชื่อ `root` และค่า `MARIADB_ROOT_PASSWORD` เมื่อต้องใช้สิทธิ์ผู้ดูแลเท่านั้น ระบบกำหนดปลายทางเป็น `mariadb:3306` ไว้แล้ว จึงไม่ต้องกรอกโฮสต์ฐานข้อมูลในหน้าเข้าสู่ระบบ

**ผ่านโปรแกรมจัดการฐานข้อมูลหรือแอปพลิเคชันบนเครื่องโฮสต์:**

| รายการ | ค่า |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `3306` |
| Database | ค่าของ `MARIADB_DATABASE` |
| Username | ค่าของ `MARIADB_USER` |
| Password | ค่าของ `MARIADB_PASSWORD` |

**จากคอนเทนเนอร์อื่น:** เชื่อมคอนเทนเนอร์เข้ากับเครือข่าย Docker เดียวกัน แล้วใช้โฮสต์ `mariadb` และพอร์ต `3306` โดย `localhost` ภายในคอนเทนเนอร์หมายถึงตัวคอนเทนเนอร์นั้นเอง ตามปกติ Compose จะเติมชื่อโปรเจกต์ไว้ด้านหน้าชื่อเครือข่ายและ volume ที่สร้างจริง

**ผ่าน MariaDB command-line client:**

```sh
docker compose exec mariadb mariadb -u root -p
```

กรอกรหัสผ่าน root เมื่อระบบร้องขอ และใช้คำสั่ง `exit` เพื่อออกจาก SQL client

### ขีดจำกัดของ phpMyAdmin

Compose กำหนดตัวแปรสภาพแวดล้อมให้ phpMyAdmin ดังนี้:

| การตั้งค่า | ค่า | ความหมาย |
| --- | --- | --- |
| `PMA_HOST` | `mariadb` | โฮสต์ฐานข้อมูลภายในเครือข่าย Docker |
| `PMA_PORT` | `3306` | พอร์ตฐานข้อมูลภายใน |
| `UPLOAD_LIMIT` | `750M` | ค่าจำกัดขนาดไฟล์อัปโหลด |
| `MAX_EXECUTION_TIME` | `5000` | ค่าเวลาประมวลผลของ PHP หน่วยวินาที |
| `MEMORY_LIMIT` | `1000M` | ค่าจำกัดหน่วยความจำของ PHP |

ค่าเหล่านี้เป็นการตั้งค่าของ PHP/phpMyAdmin ไม่ใช่ขีดจำกัดทรัพยากรของ Docker การนำเข้าไฟล์ขนาดใหญ่อาจยังล้มเหลวจาก timeout อื่นหรือทรัพยากรที่ไม่เพียงพอ ส่วน `max_input_time` ใน `custome.ini` ยังไม่มีผลตามการตั้งค่า Compose ปัจจุบัน

หลังแก้ไขตัวแปรของบริการใน Compose ให้ใช้คำสั่งนี้เพื่อสร้างบริการที่ได้รับผลกระทบใหม่:

```sh
docker compose up -d phpmyadmin
```

### คำสั่งที่ใช้บ่อย

| การทำงาน | คำสั่ง |
| --- | --- |
| ดูสถานะบริการ | `docker compose ps` |
| ติดตาม log ของทุกบริการ | `docker compose logs -f` |
| ติดตาม log ของ phpMyAdmin | `docker compose logs -f phpmyadmin` |
| หยุดคอนเทนเนอร์โดยไม่ลบ | `docker compose stop` |
| เริ่มคอนเทนเนอร์ที่หยุดไว้ | `docker compose start` |
| รีสตาร์ตบริการ | `docker compose restart` |
| ลบคอนเทนเนอร์และเครือข่ายของ Compose โดยเก็บข้อมูลฐานข้อมูลไว้ | `docker compose down` |
| สร้างบริการใหม่ตามการตั้งค่าที่เปลี่ยน | `docker compose up -d` |

### การเก็บข้อมูลและการสำรองข้อมูล

ไฟล์ฐานข้อมูลเก็บอยู่ใน named volume ชื่อ `mariadb_data` ข้อมูลยังคงอยู่เมื่อรีสตาร์ต สร้างคอนเทนเนอร์ใหม่ หรือใช้ `docker compose down` ตามปกติ แต่ volume เป็นพื้นที่เก็บข้อมูลถาวร ไม่ใช่ข้อมูลสำรอง

หากต้องการสำรองข้อมูลด้วยตนเอง ให้เปิด phpMyAdmin เลือกฐานข้อมูล เลือก **Export** ใช้รูปแบบ SQL แล้วดาวน์โหลดไฟล์ เมื่อต้องการกู้คืน ให้เลือกฐานข้อมูลปลายทางและใช้ **Import** กับไฟล์ SQL ควรเก็บไฟล์สำรองไว้นอก Docker และทดสอบว่าสามารถกู้คืนได้ โดยเฉพาะก่อนอัปเกรดอิมเมจ

หากต้องการเริ่มต้นโปรเจกต์ใหม่ทั้งหมด คำสั่งต่อไปนี้จะลบ volume ของฐานข้อมูลและ **ลบข้อมูลภายในอย่างถาวร** ให้สำรองข้อมูลที่ต้องการเก็บก่อน:

```sh
docker compose down --volumes
docker compose up -d
```

การเริ่มทำงานครั้งถัดไปจะสร้างฐานข้อมูลใหม่ตามค่า `.env` ปัจจุบัน

### การแก้ปัญหาและข้อควรพิจารณาก่อนใช้งานจริง

- **พอร์ตถูกใช้งานแล้ว:** หยุดบริการที่ใช้พอร์ตนั้น หรือเปลี่ยนพอร์ตฝั่งโฮสต์ เช่น `3307:3306` หรือ `8081:80` โปรแกรมบนโฮสต์ต้องใช้พอร์ตใหม่ แต่ phpMyAdmin ยังเชื่อมต่อภายในผ่าน `mariadb:3306`
- **ชื่อคอนเทนเนอร์ซ้ำ:** ชื่อที่กำหนดตายตัวคือ `mariadb` และ `phpmyadmin` อาจชนกับคอนเทนเนอร์เดิมหรือโปรเจกต์อีกชุด ให้เปลี่ยนหรือนำ `container_name` ออกเมื่อต้องรันหลายชุด
- **เชื่อมต่อไม่ได้ทันทีหลังเริ่มระบบ:** ตรวจสอบ `docker compose logs mariadb` และรอให้การสร้างฐานข้อมูลเสร็จสิ้น
- **เข้าสู่ระบบไม่ได้หลังแก้ `.env`:** ข้อมูลบัญชีเดิมยังอยู่ใน volume ให้ใช้บัญชีผู้ดูแลเดิมเพื่อแก้ไขผู้ใช้ อย่าลบ volume เว้นแต่ต้องการล้างข้อมูลทั้งหมด
- **แก้การตั้งค่า PHP แล้วไม่มีผล:** ไฟล์ `custome.ini` ยังไม่ได้ mount ต้องเพิ่มการ mount ไฟล์ไปยังตำแหน่งตั้งค่า PHP ที่เหมาะสมก่อน
- **การเข้าถึงผ่านเครือข่าย:** การแมปพอร์ตปัจจุบันไม่ได้จำกัดไว้ที่ localhost หากต้องการใช้งานเฉพาะบนเครื่อง ให้เปลี่ยนเป็น `127.0.0.1:3306:3306` และ `127.0.0.1:8080:80`
- **การใช้งานจริง:** โปรเจกต์ยังไม่ได้ตั้งค่า HTTPS การสำรองข้อมูลอัตโนมัติ health check หรือขีดจำกัดทรัพยากร Docker ควรตรวจสอบสิทธิ์การเข้าถึง การจัดการข้อมูลลับ ขั้นตอนสำรองข้อมูล และการกำหนดเวอร์ชันอิมเมจให้แน่นอนก่อนนำไปใช้งาน โดย URL ของ phpMyAdmin ที่ให้ไว้ใช้ HTTP
