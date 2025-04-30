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
