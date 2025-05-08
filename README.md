### **dungeon.c**

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

---

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

