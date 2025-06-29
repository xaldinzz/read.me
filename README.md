# Cella's manhwa

## Penjelasan tentang soal cella's Manhwa:
 Membuat sistem otomatis untuk:
1. Mengambil data manhwa dari API.
2. Menyimpan data ke file .txt.
3. Mengunduh gambar heroine.
4. Mengarsipkan data dan gambar ke dalam file .zip.
Semua ini disusun dengan rapi dan terurut berdasarkan nama file dan bulan rilis.

## penjelasan Kode Cella's Manhwa
### Struktur Data
Kita menggunakan struct Manhwa untuk menyimpan data per manhwa:
```
typedef struct {
    char *id;
    char *title;
    char *heroine;
    char *image_url;
    int release_month;
} Manhwa;
```
Fungsinya:
id: ID dari MyAnimeList.
title: Judul manhwa.
heroine: Nama karakter utama perempuan.
image_url: Link gambar heroine.
release_month: Bulan rilis manhwa.

### Mengganti Spasi dengan Underscore
```
char* clean(const char* title) {
    char* result = malloc(256);
    int j = 0;
    for (int i = 0; title[i]; ++i) {
        if (isalnum(title[i])) result[j++] = title[i];
        else if (isspace(title[i])) result[j++] = '_';
    }
    result[j] = '\0';
    return result;
}
```
Fungsi:
Alokasi memori untuk string hasil.
Loop setiap karakter pada title:
Jika huruf/angka → disalin ke hasil.
Jika spasi → diganti _.
Karakter lain (simbol/punctuation) → dibuang.
Tambahkan \0 di akhir untuk penutup string.
Kembalikan string yang sudah dibersihkan.

### Ambil huruf kapital
```
char* inisial(const char* title) {
    char* result = malloc(10);
    int j = 0;
    for (int i = 0; title[i];) {
        if (isalpha(title[i])) {
            result[j++] = toupper(title[i]);
            while (title[i] && title[i] != ' ') i++;
        } else {
            while (title[i] && title[i] != ' ') i++;
        }
        while (title[i] == ' ') i++;
    }
    result[j] = '\0';
    return result;
}
```
Fungsi:
Alokasikan memori untuk hasil (max 10 huruf).
Loop setiap kata.
Ambil huruf pertama (jika huruf), lalu ubah ke huruf kapital.
Lewati sisa huruf di kata itu sampai ketemu spasi.
Lanjut ke kata berikutnya.
Tambahkan \0 di akhir string dan kembalikan hasil.
### Membuat Folder
```
void make_folder(const char* path) {
    if (fork() == 0) {
        char* args[] = {"mkdir", "-p", (char*)path, NULL};
        execvp("mkdir", args);
        perror("mkdir");
        exit(1);
    } else wait(NULL);
}
```
Fungsinya: 
Fork proses baru.
Di child process: jalankan perintah mkdir -p <path> menggunakan execvp.
Jika execvp gagal, tampilkan error dengan perror().
Parent process: menunggu child selesai dengan wait().

### Ambil data dari link yang diberikan
```
void fetch_json(const char* id, const char* outfile) {
    char url[256], fileout[256];
    sprintf(url, "https://api.jikan.moe/v4/manga/%s", id);
    sprintf(fileout, "Manhwa/%s_raw.json", outfile);

    if (fork() == 0) {
        char* args[] = {"curl", "-s", "-o", fileout, url, NULL};
        execvp("curl", args);
        perror("curl");
        exit(1);
    }
    wait(NULL);
}
```
tujuan:
Mengambil (download) data manhwa dari API Jikan menggunakan curl, lalu menyimpannya sebagai file JSON mentah (.json).
### Membaca link tadi dengan jq dan tuliskan hasil parsing ke file
```
void print_jq(FILE* fout, const char* label, const char* jq_query, const char* raw) {
    char command[512];
    sprintf(command, "jq -r '%s' %s", jq_query, raw);
    FILE* fp = popen(command, "r");

    fprintf(fout, "%s", label);
    if (fp) {
        char line[512];
        if (fgets(line, sizeof(line), fp)) {
            line[strcspn(line, "\n")] = 0;
            fprintf(fout, "%s\n", line);
        } else {
            fprintf(fout, "-\n");
        }
        pclose(fp);
    } else {
        fprintf(fout, "-\n");
    }
}
```
Penjelasannya:
Bentuk perintah jq seperti:
```
jq -r '<query>' <nama_file_json>
```
Jalankan perintah tersebut dengan popen(), untuk membaca hasil output jq.
Tulis label (misal "Title: ").
Jika output jq berhasil dibaca:
Ambil baris pertama → hapus newline → tulis ke file .txt.
Jika gagal atau kosong → tulis "-".

### ambil data dari link yang diberikan tadi
```
    print_jq(fout, "Title: ", ".data.title // \"-\"", raw);
    print_jq(fout, "Status: ", ".data.status // \"-\"", raw);
    print_jq(fout, "Release: ", ".data.published.from // \"-\"", raw);
    print_jq(fout, "Genre: ", ".data.genres | map(.name) | join(\", \") // \"-\"", raw);
    print_jq(fout, "Theme: ", ".data.themes | map(.name) | join(\", \") // \"-\"", raw);
    print_jq(fout, "Author: ", ".data.authors | map(.name) | join(\", \") // \"-\"", raw);

    fclose(fout);
    remove(raw);
}
```
Penjelasan:
Mengisi file .txt dengan informasi metadata manhwa dari file JSON menggunakan jq, lalu menghapus file JSON mentah. 
fclose(fout): Menutup file .txt setelah semua data ditulis.
remove(raw): Menghapus file JSON mentah karena sudah tidak dibutuhkan.
### Download Gambar Heroine 
```
void* download_images(void* arg) {
    ThreadArgs* data = (ThreadArgs*)arg;
    for (int i = 1; i <= data->count; i++) {
        char filename[128], cmd[512];
        sprintf(filename, "Heroines/%s/%s_%02d.jpg", data->name, data->name, i);
        sprintf(cmd, "wget --user-agent=\"Mozilla\" -q \"%s\" -O \"%s\"", data->url, filename);
        system(cmd);
        printf("Downloaded: %s\n", filename);
    }
    return NULL;
}
```
Cara Kerja:
Cast argumen thread ke ThreadArgs untuk mengambil:
name heroine
url gambar
count jumlah gambar
Loop dari 1 hingga count:
Buat nama file dengan format:
```
Heroines/<heroine>/<heroine>_01.jpg, _02.jpg, dst
```
Buat perintah wget untuk mengunduh dari URL heroine.
Jalankan system(cmd) untuk mengeksekusi unduhan.
Tampilkan nama file yang telah diunduh.
Setelah selesai, kembalikan NULL ke thread caller.

### Zip gambar Heroine
```
void zip(const char* initials, const char* heroine) {
    char zipname[256], cmd[1024];
    sprintf(zipname, "Archive/Images/%s_%s.zip", initials, heroine);
    sprintf(cmd, "find Heroines/%s -type f -name '*.jpg' | sort | zip -@ \"%s\"", heroine, zipname);

    if (fork() == 0) {
        execl("/bin/sh", "sh", "-c", cmd, NULL);
        perror("zip");
        exit(1);
    }
    wait(NULL);
}
```
find: mencari semua file .jpg heroine.
sort: mengurutkan nama file (agar ZIP-nya rapi).
zip -@: membaca daftar file dari input (stdin) dan membuat ZIP.
Jalankan perintah di child process menggunakan:
```
execl("/bin/sh", "sh", "-c", cmd, NULL);
```
Menjalankan seluruh cmd sebagai perintah shell (sh -c).
Parent akan menunggu proses selesai dengan wait(NULL).

### Hapus gambar setelah di zip
```
void delete(const char* heroine) {
    char cmd[256];
    sprintf(cmd, "find Heroines/%s -type f -name '*.jpg' -delete", heroine);
    system(cmd);
}
```
Penjelasan :
find: mencari semua file.
-type f: hanya cari file (bukan folder).
-name '*.jpg': filter hanya file .jpg.
-delete: langsung hapus file yang ditemukan.

### Menzip File TXT 
```
void ziptxt(const char* outfile, const char* initials) {
    char zipname[256], path[256];
    sprintf(zipname, "Archive/%s.zip", initials);
    sprintf(path, "Manhwa/%s.txt", outfile);

    if (fork() == 0) {
        char* args[] = {"zip", "-j", zipname, path, NULL};
        execvp("zip", args);
        perror("zip");
        exit(1);
    }
    wait(NULL);
}
```
Penjealsannya: 
Jalankan perintah zip -j menggunakan fork dan exec:
```
zip -j Archive/<INISIAL>.zip Manhwa/<NAMA_FILE>.txt
```
-j: (junk path) menyimpan file dalam ZIP tanpa struktur folder.

Proses dilakukan di child (fork()), eksekusi dengan execvp(), dan ditunggu oleh parent dengan wait(NULL).

### kode Memanggil task 3A
```
    for (int i = 0; i < n; i++) {
        filenames[i] = clean(list[i].title);
        fetch_json(list[i].id, filenames[i]);
        write_text(filenames[i]);
    }
```
### Kode memanggil Task 3B
```
    for (int i = 0; i < n; i++) {
        char* initials = inisial(list[i].title);
        ziptxt(filenames[i], initials);
        free(initials);
    }
```
### Kode memanggil Task 3C ( Gambar Heroine)
```
    for (int i = 0; i < n; i++) {
        char foldername[64];
        sprintf(foldername, "Heroines/%s", list[i].heroine);
        make_folder(foldername);

        ThreadArgs* arg = malloc(sizeof(ThreadArgs));
        strcpy(arg->name, list[i].heroine);
        strcpy(arg->url, list[i].image_url);
        arg->count = list[i].release_month;

        pthread_t tid;
        pthread_create(&tid, NULL, download_images, arg);
        pthread_join(tid, NULL);

        free(arg);
    }
```
### Kode memanggil Task 3D (Zip Lalu Hapus)
```
    for (int i = 0; i < n; i++) {
        char* initials_now = inisial(list[i].title);
        zip(initials_now, list[i].heroine);
        delete(list[i].heroine);
        free(initials_now);
        free(filenames[i]);
    }
```
## Output
### Log
Log yang akan muncul ketika kita mengcompile file tersebut akan muncul seperti ini:
```
updating: Mistaken_as_the_Monster_Dukes_Wife.txt (deflated 17%)
updating: The_Villainess_Lives_Again.txt (deflated 16%)
updating: No_I_Only_Charmed_the_Princess.txt (deflated 15%)
updating: Darling_Why_Cant_We_Divorce.txt (deflated 16%)
Downloaded: Heroines/Lia/Lia_01.jpg
Downloaded: Heroines/Lia/Lia_02.jpg
Downloaded: Heroines/Lia/Lia_03.jpg
Downloaded: Heroines/Tia/Tia_01.jpg
Downloaded: Heroines/Tia/Tia_02.jpg
Downloaded: Heroines/Tia/Tia_03.jpg
Downloaded: Heroines/Tia/Tia_04.jpg
Downloaded: Heroines/Tia/Tia_05.jpg
Downloaded: Heroines/Tia/Tia_06.jpg
Downloaded: Heroines/Tia/Tia_07.jpg
Downloaded: Heroines/Yeldia/Yeldia_01.jpg
Downloaded: Heroines/Yeldia/Yeldia_02.jpg
Downloaded: Heroines/Yeldia/Yeldia_03.jpg
Downloaded: Heroines/Yeldia/Yeldia_04.jpg
Downloaded: Heroines/Ophelia/Ophelia_01.jpg
Downloaded: Heroines/Ophelia/Ophelia_02.jpg
Downloaded: Heroines/Ophelia/Ophelia_03.jpg
Downloaded: Heroines/Ophelia/Ophelia_04.jpg
Downloaded: Heroines/Ophelia/Ophelia_05.jpg
Downloaded: Heroines/Ophelia/Ophelia_06.jpg
Downloaded: Heroines/Ophelia/Ophelia_07.jpg
Downloaded: Heroines/Ophelia/Ophelia_08.jpg
Downloaded: Heroines/Ophelia/Ophelia_09.jpg
Downloaded: Heroines/Ophelia/Ophelia_10.jpg
updating: Heroines/Lia/Lia_01.jpg (deflated 1%)
updating: Heroines/Lia/Lia_02.jpg (deflated 1%)
updating: Heroines/Lia/Lia_03.jpg (deflated 1%)
updating: Heroines/Tia/Tia_01.jpg (deflated 0%)
updating: Heroines/Tia/Tia_02.jpg (deflated 0%)
updating: Heroines/Tia/Tia_03.jpg (deflated 0%)
updating: Heroines/Tia/Tia_04.jpg (deflated 0%)
updating: Heroines/Tia/Tia_05.jpg (deflated 0%)
updating: Heroines/Tia/Tia_06.jpg (deflated 0%)
updating: Heroines/Tia/Tia_07.jpg (deflated 0%)
updating: Heroines/Yeldia/Yeldia_01.jpg (deflated 3%)
updating: Heroines/Yeldia/Yeldia_02.jpg (deflated 3%)
updating: Heroines/Yeldia/Yeldia_03.jpg (deflated 3%)
updating: Heroines/Yeldia/Yeldia_04.jpg (deflated 3%)
updating: Heroines/Ophelia/Ophelia_01.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_02.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_03.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_04.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_05.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_06.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_07.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_08.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_09.jpg (deflated 5%)
updating: Heroines/Ophelia/Ophelia_10.jpg (deflated 5%)
selesai deh, cella bilang makasih ya ke kamu!!!![1] + Done
```

## Task 3A
Untuk 3A kita disuruh menamapilkan File Txt yang berisi data yang diambil link tadi berikut ini adalah contoh isi dari File TXT:

![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2015-47-10.png?raw=true)

## Task 3B
Untuk 3B Kita disuruh untuk membuat file zip yang berisi data dari file txt ini adalah contoh file zipnya:
![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2015-55-54.png)

Bisa di liat ini adalah screenshot yang berisi zip yang di dalamnya berada File Txtnya. dan juga sesuai dengan perintah di Task 3B kita harus membuat file yang berisi hanya huruf awal dengan huruf kapitalnya saja.

## Task 3C
Untuk 3C kita disuruh untuk mendownload Gambar heroine yang kita dapat dari internet, Ini adalah contoh yang akan muncul di file ketika kita sedang merun code ini:

![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2016-01-23.png)
![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2016-02-52.png)

ini adalah salah satu contoh yang berada di file dan berisi gambar yang sudah di download, dan juga mendownload sejumlah bulan release manhwa tersebut

## task 3D
Untuk 3D kita disuruh untuk menzip Gambar yang sudah di buat dan menghapus gambar yang berada di file heroine. berikut contoh di file tersebut:

![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2016-12-23.png)

![imagealt](https://github.com/xaldinzz/read.me/blob/main/Screenshot%20from%202025-04-30%2016-11-56.png)

Berikut adalah foto-foto yang membuktikan bahwa kode ini bisa berjalan dengan benar.




