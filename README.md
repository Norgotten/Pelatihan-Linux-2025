Pada soal ini, diminta untuk mengimplementasikan sebuah permainan. Permainan ini menggunakan arsitektur **client-server** di mana `dungeon.c` berfungsi sebagai server yang menangani logika permainan, sedangkan `player.c` bertindak sebagai client yang digunakan pemain untuk berinteraksi dengan dunia dungeon.

Permainan ini memiliki beberapa fitur utama, yaitu:

* **Menjelajahi Dungeon:** Pemain dapat mengeksplorasi dunia dungeon dan berinteraksi dengan berbagai elemen, seperti toko senjata dan musuh.
* **Toko Senjata:** Pemain dapat membeli senjata dengan berbagai kemampuan passive yang unik.
* **Inventori:** Senjata yang dibeli dapat disimpan dan digunakan melalui inventori.
* **Battle Mode:** Pemain dapat memasuki mode pertempuran untuk melawan musuh dengan sistem health bar dan damage.

Sebagian besar **permasalahan/error** yang saya alami ketika mengerjakan soal ini ada pada `bagian battle mode`, seperti health bar yang terkadang formatnya error, dan tidak muncul tiba tiba. Juga input yang terkadang tidak terbaca, dan loop untuk input yang berhenti tiba-tiba ditengah pertarungan battlemode. 

---

### **dungeon.c**


### **1. Inisialisasi Server**

```c
int main() {
    int sd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in a = {AF_INET, htons(PORT), INADDR_ANY};
    bind(sd, (struct sockaddr*)&a, sizeof(a));
    listen(sd, 3);
    printf("Dungeon server berjalan di port %d...\n", PORT);
    
    while (1) {
        int cs = accept(sd, NULL, NULL);
        if (cs >= 0) handle_client(cs);
    }
    return 0;
}
```

- `socket(AF_INET, SOCK_STREAM, 0)`  
  - Membuat socket TCP. `AF_INET` menunjukkan penggunaan IPv4, `SOCK_STREAM` untuk komunikasi berbasis koneksi, dan `0` untuk protokol default (TCP).

- `bind()`  
  - Mengaitkan socket dengan alamat dan port yang ditentukan (`PORT 3010`). Port ini akan menjadi endpoint server.

- `listen(sd, 3)`  
  - Mengatur socket agar siap menerima koneksi. `3` menunjukkan jumlah maksimum antrian koneksi yang dapat diterima secara bersamaan.

- `accept()`  
  - Menerima koneksi dari client. Jika berhasil (`cs >= 0`), maka koneksi tersebut akan diproses melalui `handle_client()`.


### **2. Struktur Data Player**

```c
typedef struct {
    int gold;
    int base_damage;
    char equipped[MAX_NAME];
    Weapon inv[MAX_INV];
    int inv_count;
    int kills;
} Player;
```

- **gold**: Menyimpan jumlah uang yang dimiliki oleh pemain. Defaultnya adalah 500 gold saat awal permainan.
- **base_damage**: Damage dasar pemain tanpa senjata. Defaultnya adalah 5 damage.
- **equipped**: Nama senjata yang sedang dipakai. Awalnya adalah "Fists".
- **inv[]**: Array dari tipe `Weapon` yang menyimpan senjata yang dimiliki oleh pemain. Maksimal 10 senjata.
- **inv_count**: Jumlah senjata dalam inventori. Dimulai dari `1` (hanya "Fists").
- **kills**: Jumlah musuh yang telah dikalahkan pemain. Dimulai dari `0`.


### **3. Utility Functions**

#### 3.1 **rand_range()**
```c
int rand_range(int min, int max) {
    return rand() % (max - min + 1) + min;
}
```

- Fungsi ini menghasilkan bilangan acak antara `min` dan `max`.
- Menggunakan modulus (`%`) untuk memastikan nilai tetap dalam rentang tersebut.
- Digunakan untuk menentukan **HP musuh** dan **reward gold**.


#### 3.2 **print_healthbar()**
```c
void print_healthbar(int hp, int max_hp, char* bar) {
    int len = 20;
    int filled = hp * len / max_hp;

    char temp[64] = "";
    for (int i = 0; i < filled; i++) strcat(temp, "#");

    char empty[64] = "";
    for (int i = filled; i < len; i++) strcat(empty, "-");

    sprintf(bar, "[%s%s]\n", temp, empty);
}
```

- Fungsi untuk mencetak health bar dengan panjang tetap (`len = 20`).
- **filled**: Jumlah karakter `#` yang merepresentasikan HP saat ini.
- **empty**: Karakter `-` yang menunjukkan sisa HP yang telah hilang.

Contoh:
- Jika `hp = 10` dan `max_hp = 20`, maka health bar akan terlihat seperti:  
  ```
  [##########----------]
  ```


### **4. Battle Mode - handle_battle()**

```c
void handle_battle(Player* P, int sock) {
    int monster_hp = rand_range(50, 200);
    int monster_max = monster_hp;
    char msg[BUF_SZ], bar[64];

    print_healthbar(monster_hp, monster_max, bar);
    snprintf(msg, sizeof(msg), "Musuh HP: %s (%d/%d)\n", bar, monster_hp, monster_max);
    send(sock, msg, strlen(msg), 0);

    while (1) {
        char input[64] = {0};
        int r = read(sock, input, sizeof(input) - 1);
        input[strcspn(input, "\r\n")] = 0;

        if (!strcmp(input, "exit")) {
            send(sock, "Keluar dari Battle Mode.\n", 26, 0);
            break;
        } else if (!strcmp(input, "attack")) {
            int dmg = P->base_damage + rand_range(1, 4);
            int crit = (rand() % 100 < 15) ? 2 : 1;
            int total_dmg = dmg * crit;

            monster_hp -= total_dmg;

            if (monster_hp <= 0) {
                int reward = rand_range(50, 100);
                P->gold += reward;
                P->kills++;
                snprintf(msg, sizeof(msg), "Musuh dikalahkan! Anda mendapat %d gold.\n", reward);
                send(sock, msg, strlen(msg), 0);
                break;
            }
        }
    }
}
```

- **Inisialisasi Musuh:**
  - HP musuh di-generate secara acak (`50-200`).
  - Health bar di-generate menggunakan `print_healthbar()`.

- **Input Pemain:**
  - Jika `exit`, maka keluar dari Battle Mode.
  - Jika `attack`, pemain menyerang musuh.

- **Damage Calculation:**
  - **Base Damage** ditambah acak (`1-4`).
  - **Critical Hit:** 15% kemungkinan double damage (`crit = 2`).

- **Reward & Kills:**
  - Jika musuh kalah, pemain akan mendapat reward gold (`50-100`).
  - Jumlah kill bertambah (`P->kills++`).


### **5. Weapon Shop - show_shop()**

```c
void show_shop(int sock) {
    char resp[BUF_SZ];
    strcpy(resp, "=== Weapon Shop ===\n");
    
    for (int i = 0; i < get_weapon_count(); i++) {
        Weapon w = get_weapon(i);
        char line[256];
        snprintf(line, sizeof(line), "%d. %s - %d gold, %d dmg\n", i + 1, w.name, w.price, w.damage);
        strcat(resp, line);
    }

    send(sock, resp, strlen(resp), 0);
}
```

- Menampilkan senjata yang tersedia di toko.
- Data senjata diambil dari `shop.c` melalui fungsi `get_weapon()`.


### **6. Inventory Management - handle_inventory()**

```c
void handle_inventory(Player* p, int sock) {
    char resp[BUF_SZ];
    strcpy(resp, "=== YOUR INVENTORY ===\n");

    for (int i = 0; i < p->inv_count; i++) {
        Weapon w = p->inv[i];
        char line[256];
        snprintf(line, sizeof(line), "[%d] %s | %ddmg\n", i + 1, w.name, w.damage);
        strcat(resp, line);
    }

    send(sock, resp, strlen(resp), 0);
}
```

- Menampilkan semua senjata yang dimiliki pemain.
- Jika senjata memiliki passive, maka informasi tersebut juga akan ditampilkan.


### **7. Equip Weapon - handle_equip()**

```c
else if (!strncmp(buf, "EQUIP ", 6)) {
    int idx = atoi(buf + 6) - 1;
    char resp[BUF_SZ];

    if (idx >= 0 && idx < P->inv_count) {
        Weapon w = P->inv[idx];
        strcpy(P->equipped, w.name);
        P->base_damage = w.damage;
        snprintf(resp, sizeof(resp), "Senjata %s telah di-equip.\n", w.name);
    } else {
        snprintf(resp, sizeof(resp), "Pilihan senjata tidak valid.\n");
    }

    send(sock, resp, strlen(resp), 0);
}
```

- Memproses perintah `EQUIP`.
- Jika senjata ada di inventori (`idx < inv_count`), maka senjata tersebut akan dipakai (`equipped`).


### **8. Player Status - show_stats()**

```c
void show_stats(Player* p, int sock) {
    char resp[BUF_SZ];
    snprintf(resp, sizeof(resp),
        "Gold: %d\nEquipped Weapon: %s\nBase Damage: %d\nKills: %d\n",
        p->gold, p->equipped, p->base_damage, p->kills);

    send(sock, resp, strlen(resp), 0);
}
```

- Menampilkan status pemain, termasuk gold, senjata yang dipakai, base damage, dan jumlah kill.

---

### **player.c**


### **1. Inisialisasi Client**

```c
int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in srv = { .sin_family = AF_INET, .sin_port = htons(PORT) };
    inet_pton(AF_INET, IP, &srv.sin_addr);

    if (connect(sock, (struct sockaddr*)&srv, sizeof(srv)) < 0) {
        perror("Gagal koneksi ke server");
        return 1;
    }

    int choice;
    char cmd[64];
```

- `socket(AF_INET, SOCK_STREAM, 0)`  
  - Membuat socket untuk komunikasi TCP.

- `inet_pton(AF_INET, IP, &srv.sin_addr)`  
  - Mengubah alamat IP dari string (`127.0.0.1`) menjadi format biner.

- `connect(sock, (struct sockaddr*)&srv, sizeof(srv))`  
  - Menghubungkan client ke server. Jika koneksi gagal, maka program akan keluar.


### **2. Fungsi recv_print()**

```c
void recv_print(int s) {
    char buf[BSZ + 1];
    int n = read(s, buf, BSZ);
    if (n > 0) {
        buf[n] = '\0';
        printf("%s\n", buf);
    }
}
```

- **Fungsi Utama:** Menerima data dari server dan mencetaknya ke layar.

- **Buffer `buf`** digunakan untuk menampung data sementara yang dibaca dari socket:
  - Karena data dari server bisa panjang dan tidak langsung ditampilkan ke layar satu per satu, maka digunakan buffer (penampung sementara) untuk menyimpan semua data yang dibaca dari socket agar bisa diproses sekaligus.
  - `BSZ` (2048 byte) adalah batas maksimal ukuran data yang bisa diterima dalam satu kali baca.
  - Setelah data diterima dan diakhiri dengan `\0` (null-terminator), `printf()` mencetak seluruh isi buffer ke layar.


### **3. Main Menu Loop**

```c
while (1) {
    printf("\n\x1b[95m=== TLD Menu ===\x1b[0m\n");
    printf("\x1b[96m[1]\x1b[0m Status Diri\n");
    printf("\x1b[96m[2]\x1b[0m Toko Senjata\n");
    printf("\x1b[96m[3]\x1b[0m Periksa Inventori & Atur Perlengkapan\n");
    printf("\x1b[96m[4]\x1b[0m Battle Mode\n");
    printf("\x1b[96m[5]\x1b[0m Keluar dari TLD\n");    
    printf("\x1b[96mChoose:\x1b[0m ");
    
    scanf("%d", &choice);
    getchar();  // Membersihkan newline di buffer
```

- Menu utama terdiri dari 5 opsi:
  1. Status Diri (`SHOW_STATS`)
  2. Toko Senjata (`SHOW_SHOP`)
  3. Inventori (`INVENTORY` dan `EQUIP`)
  4. Battle Mode (`BATTLE`)
  5. Keluar (`EXIT`)


### **4. Menampilkan Status Pemain (SHOW_STATS)**

```c
if (choice == 1) {
    strcpy(cmd, "SHOW_STATS");
    send(sock, cmd, strlen(cmd), 0);
    recv_print(sock);
}
```

- Mengirim perintah `"SHOW_STATS"` ke server.
- Menerima respons dari server menggunakan `recv_print()`.


### **5. Toko Senjata (SHOW_SHOP & BUY)**

```c
else if (choice == 2) {
    strcpy(cmd, "SHOW_SHOP");
    send(sock, cmd, strlen(cmd), 0);
    recv_print(sock);

    printf("\x1b[96mPilih senjata untuk dibeli (1-5), 0 untuk batal:\x1b[0m ");
    int b;
    scanf("%d", &b);
    getchar();

    if (b >= 1 && b <= 5) {
        snprintf(cmd, sizeof(cmd), "BUY %d", b);
        send(sock, cmd, strlen(cmd), 0);
        recv_print(sock);
    } else {
        printf("Batal atau pilihan tidak valid.\n");
    }
}
```

- Pemain melihat daftar senjata yang tersedia dan dapat membeli jika cukup gold.


### **6. Inventori & Equip Weapon (INVENTORY & EQUIP)**

```c
else if (choice == 3) {
    strcpy(cmd, "INVENTORY");
    send(sock, cmd, strlen(cmd), 0);
    recv_print(sock);

    printf("\x1b[96mPilih indeks senjata untuk equip (1-n), 0 untuk batal:\x1b[0m ");
    int e;
    scanf("%d", &e);
    getchar();

    if (e > 0) {
        snprintf(cmd, sizeof(cmd), "EQUIP %d", e);
        send(sock, cmd, strlen(cmd), 0);
        recv_print(sock);
    } else {
        printf("Batal equip.\n");
    }
}
```

- Pemain dapat melihat inventori dan memilih senjata untuk digunakan.


### **7. Battle Mode (BATTLE)**

```c
else if (choice == 4) {
    strcpy(cmd, "BATTLE");
    send(sock, cmd, strlen(cmd), 0);
    recv_print(sock);

    while (1) {
        char bcmd[64] = {0};
        printf("\x1b[96mBattle> \x1b[0m");
        scanf("%s", bcmd);
        getchar();

        send(sock, bcmd, strlen(bcmd), 0);
        if (!strcmp(bcmd, "exit")) {
            recv_print(sock);
            break;
        }
        recv_print(sock);
    }
}
```

- Pemain masuk ke mode pertempuran dan dapat mengetik `attack` atau `exit`.
- Respons akan ditampilkan langsung melalui `recv_print()`.


### **8. Keluar dari Program (EXIT)**

```c
else if (choice == 5) {
    strcpy(cmd, "EXIT");
    send(sock, cmd, strlen(cmd), 0);
    printf("\x1b[33mGoodbye, adventurer!\x1b[0m\n");
    break;
}
```

- Mengirim perintah `"EXIT"` dan keluar dari permainan.


### **9. Error Handling**

- Input menu yang tidak valid akan menghasilkan pesan:
  ```
  Pilihan tidak valid.
  ```

- Battle Mode juga menangani input salah dengan memberi pesan dari server.

---

### **shop.c**


### **1. Struktur Data Weapon**

```c
#define MAX_WEAPONS 5
#define MAX_NAME 32

typedef struct {
    char name[MAX_NAME];
    int price;
    int damage;
    char passive[64];
} Weapon;
```

- **MAX_WEAPONS**: Jumlah maksimal senjata di toko (`5` senjata).
- **MAX_NAME**: Panjang maksimal nama senjata (`32` karakter).

**Struktur Weapon** terdiri dari:
- `name`: Nama senjata (contoh: "Dark Repulsor").
- `price`: Harga senjata dalam gold.
- `damage`: Damage dasar yang diberikan senjata.
- `passive`: Passive skill yang dimiliki senjata. Jika tidak ada passive, maka string kosong (`""`).

---

### **2. Inisialisasi Senjata di Toko**

```c
Weapon shopWeapons[MAX_WEAPONS] = {
    {"Dark Repulsor", 200, 13, ""},
    {"Elucidator", 200, 12, ""},
    {"Gilta Brille", 250, 14, ""},
    {"Mercenary Twinblade", 150, 9, "Quick Slash"},
    {"Togetsu Waxing", 300, 15, "Thunder Blast"}
};
```

- **Dark Repulsor**: Harga `200 gold`, damage `13`, tanpa passive.
- **Elucidator**: Harga `200 gold`, damage `12`, tanpa passive.
- **Gilta Brille**: Harga `250 gold`, damage `14`, tanpa passive.
- **Mercenary Twinblade**: Harga `150 gold`, damage `9`, passive `"Quick Slash"`.
  - `"Quick Slash"`: Kemungkinan serangan tambahan.
- **Togetsu Waxing**: Harga `300 gold`, damage `15`, passive `"Thunder Blast"`.
  - `"Thunder Blast"`: Tambahan damage jika passive aktif.

**Penjelasan:**
- Senjata yang memiliki passive akan memberikan efek tambahan ketika passive tersebut aktif. Efek ini akan diproses saat battle di `dungeon.c`.


### **3. Fungsi get_weapon_count()**

```c
int get_weapon_count() {
    return MAX_WEAPONS;
}
```

- Fungsi ini hanya mengembalikan jumlah senjata yang ada di toko (`5` senjata).
- Digunakan di `dungeon.c` untuk menampilkan jumlah senjata saat pemain membuka toko (`SHOW_SHOP`).


### **4. Fungsi get_weapon()**

```c
Weapon get_weapon(int index) {
    return shopWeapons[index];
}
```

- Fungsi ini digunakan untuk mengambil data senjata berdasarkan indeks (`index`).
- **Parameter:**
  - `index`: Indeks senjata (`0` hingga `MAX_WEAPONS - 1`).

- **Return Value:**  
  - Mengembalikan struktur `Weapon` yang sesuai dengan indeks yang diberikan.

**Contoh Penggunaan:**
- Jika `get_weapon(1)` dipanggil, maka akan mengembalikan senjata `"Elucidator"` dengan harga `200 gold` dan damage `12`.


### **5. Mekanisme Pembelian Senjata di dungeon.c**

Meskipun fungsi pembelian senjata (`BUY`) tidak langsung diimplementasikan di `shop.c`, namun `shop.c` berfungsi sebagai **penyedia data senjata**. Berikut adalah mekanisme pembelian senjata di `dungeon.c` menggunakan fungsi dari `shop.c`:

```c
else if (!strncmp(buf, "BUY ", 4)) {
    int idx = atoi(buf + 4) - 1;
    char resp[BUF_SZ];

    if (idx < 0 || idx >= get_weapon_count()) {
        snprintf(resp, sizeof(resp), "Pilihan senjata tidak valid.\n");
    } else {
        Weapon w = get_weapon(idx);

        if (P->gold >= w.price) {
            P->gold -= w.price;
            add_inv(P, w);
            snprintf(resp, sizeof(resp), "Berhasil membeli %s!\nGold sisa: %d\n", w.name, P->gold);
        } else {
            snprintf(resp, sizeof(resp), "Gold tidak cukup untuk membeli %s.\n", w.name);
        }
    }
    send(sock, resp, strlen(resp), 0);
}
```

**Penjelasan:**
- Perintah `"BUY <index>"` diproses dengan mengambil senjata menggunakan `get_weapon()`.
- Jika `gold` pemain cukup (`P->gold >= w.price`), maka senjata tersebut akan ditambahkan ke inventori.
- Jika tidak cukup, maka pesan `"Gold tidak cukup"` akan dikirimkan ke client.


### **6. Passive Weapon Implementation**

- Senjata `"Mercenary Twinblade"` memiliki passive `"Quick Slash"`:
  - `"Quick Slash"` memungkinkan pemain menyerang dua kali secara acak.
  - Passive ini akan dicek saat battle di `dungeon.c`.

- Senjata `"Togetsu Waxing"` memiliki passive `"Thunder Blast"`:
  - `"Thunder Blast"` menambahkan `10 damage` secara acak.
  - Passive ini juga diimplementasikan di battle mode (`dungeon.c`).

---

### Revisi

Untuk revisi sendiri, saya hanya menambahkan kill count pada menu **status player**
