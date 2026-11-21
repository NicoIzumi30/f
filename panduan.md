# Panduan Pengerjaan Ujian CPMK 03 — Paket 4

**Aplikasi:** Antrian Servis Bengkel  
**Teknologi:** Flutter, Dart, Hive, GetX, MVC, Material 3, Intl  
**Target:** 6 commit Git bertahap  
**Catatan:** Seluruh potongan kode pada panduan ini ditulis **tanpa komentar dalam kode**. Lakukan commit pada akhir tiap tahap, bukan sekaligus setelah aplikasi selesai.

## Ketentuan soal

Aplikasi mencatat antrian servis motor dan mobil dengan atribut berikut.

| Field | Tipe | Ketentuan |
|---|---|---|
| `platNomor` | `String` | Wajib, huruf besar |
| `namaPemilik` | `String` | Wajib |
| `jenisKendaraan` | `String` | Motor atau Mobil |
| `keluhan` | `String` | Wajib |
| `biaya` | `double` | Minimal 0, ditampilkan sebagai Rupiah |
| `selesai` | `bool` | Default `false` |

Tambahan `id` dipakai sebagai kunci unik internal Hive. Fitur wajib: CRUD, pencarian, filter **Proses/Selesai**, tombol **Tandai Selesai** di daftar, serta total pendapatan dari servis yang selesai.

---

# Commit 1 — Inisialisasi Proyek Flutter

## 1. Membuat proyek

```bash
flutter create antrian_servis_bengkel
cd antrian_servis_bengkel
code .
flutter doctor
```

## 2. Menginstal dependency

```bash
flutter pub add get hive hive_flutter
```

## 3. Membuat struktur folder

```bash
mkdir -p lib/models lib/services lib/controllers lib/views lib/bindings lib/routes

touch lib/models/servis_model.dart
touch lib/models/servis_adapter.dart
touch lib/services/hive_service.dart
touch lib/controllers/servis_controller.dart
touch lib/views/servis_list_view.dart
touch lib/views/servis_form_view.dart
touch lib/bindings/servis_binding.dart
touch lib/routes/app_routes.dart
touch lib/routes/app_pages.dart
```

Struktur yang disiapkan:

```text
lib/
├── main.dart
├── models/
│   ├── servis_model.dart
│   └── servis_adapter.dart
├── services/
│   └── hive_service.dart
├── controllers/
│   └── servis_controller.dart
├── views/
│   ├── servis_list_view.dart
│   └── servis_form_view.dart
├── bindings/
│   └── servis_binding.dart
└── routes/
    ├── app_routes.dart
    └── app_pages.dart
```

## 4. Isi `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: 'Antrian Servis Bengkel',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorSchemeSeed: Colors.indigo,
        useMaterial3: true,
      ),
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Antrian Servis Bengkel'),
          centerTitle: true,
        ),
        body: const Center(
          child: Text('Aplikasi Antrian Servis Bengkel'),
        ),
      ),
    );
  }
}
```

## 5. Verifikasi dan commit

```bash
flutter analyze
flutter run
```

Jika `flutter create` sudah membuat commit otomatis, biarkan riwayat tersebut tetap ada. Setelah tahapan di atas selesai:

```bash
git status
git add .
git commit -m "chore: initialize Flutter project with MVC structure"
git log --oneline -5
```

**Selesai Commit 1:** proyek berjalan dan struktur MVC tersedia. File kosong akan diisi bertahap.

---

# Commit 2 — Model dan Database Hive

## 1. Isi `lib/models/servis_model.dart`

```dart
class Servis {
  final String id;
  final String platNomor;
  final String namaPemilik;
  final String jenisKendaraan;
  final String keluhan;
  final double biaya;
  final bool selesai;

  Servis({
    required this.id,
    required this.platNomor,
    required this.namaPemilik,
    required this.jenisKendaraan,
    required this.keluhan,
    required this.biaya,
    this.selesai = false,
  });

  Servis copyWith({
    String? platNomor,
    String? namaPemilik,
    String? jenisKendaraan,
    String? keluhan,
    double? biaya,
    bool? selesai,
  }) {
    return Servis(
      id: id,
      platNomor: platNomor ?? this.platNomor,
      namaPemilik: namaPemilik ?? this.namaPemilik,
      jenisKendaraan: jenisKendaraan ?? this.jenisKendaraan,
      keluhan: keluhan ?? this.keluhan,
      biaya: biaya ?? this.biaya,
      selesai: selesai ?? this.selesai,
    );
  }
}
```

`copyWith` menghasilkan objek baru, mempertahankan ID yang sama. Ini berguna ketika mengedit data maupun menandai servis selesai.

## 2. Isi `lib/models/servis_adapter.dart`

```dart
import 'package:hive/hive.dart';

import 'servis_model.dart';

class ServisAdapter extends TypeAdapter<Servis> {
  @override
  final int typeId = 0;

  @override
  Servis read(BinaryReader reader) {
    return Servis(
      id: reader.readString(),
      platNomor: reader.readString(),
      namaPemilik: reader.readString(),
      jenisKendaraan: reader.readString(),
      keluhan: reader.readString(),
      biaya: reader.readDouble(),
      selesai: reader.readBool(),
    );
  }

  @override
  void write(BinaryWriter writer, Servis obj) {
    writer.writeString(obj.id);
    writer.writeString(obj.platNomor);
    writer.writeString(obj.namaPemilik);
    writer.writeString(obj.jenisKendaraan);
    writer.writeString(obj.keluhan);
    writer.writeDouble(obj.biaya);
    writer.writeBool(obj.selesai);
  }
}
```

Urutan field pada `read()` dan `write()` harus identik. `typeId` harus unik untuk setiap adapter di aplikasi.

## 3. Isi `lib/services/hive_service.dart`

```dart
import 'package:hive/hive.dart';

import '../models/servis_model.dart';

class HiveService {
  static const String boxName = 'servis_box';

  Box<Servis> get _box => Hive.box<Servis>(boxName);

  List<Servis> getAll() => _box.values.toList();

  Future<void> create(Servis servis) {
    return _box.put(servis.id, servis);
  }

  Future<void> update(Servis servis) {
    return _box.put(servis.id, servis);
  }

  Future<void> delete(String id) {
    return _box.delete(id);
  }
}
```

`put()` menambah data ketika key belum ada, dan memperbarui data ketika key sudah ada.

## 4. Ganti `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:hive_flutter/hive_flutter.dart';

import 'models/servis_adapter.dart';
import 'models/servis_model.dart';
import 'services/hive_service.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  Hive.registerAdapter(ServisAdapter());
  await Hive.openBox<Servis>(HiveService.boxName);
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: 'Antrian Servis Bengkel',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorSchemeSeed: Colors.indigo,
        useMaterial3: true,
      ),
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Antrian Servis Bengkel'),
          centerTitle: true,
        ),
        body: const Center(
          child: Text('Aplikasi Antrian Servis Bengkel'),
        ),
      ),
    );
  }
}
```

## 5. Verifikasi dan commit

```bash
dart format lib
flutter analyze
flutter run
git add lib/main.dart lib/models/servis_model.dart lib/models/servis_adapter.dart lib/services/hive_service.dart
git commit -m "feat: implement service model and Hive database"
```

**Selesai Commit 2:** Model, adapter, service CRUD, dan inisialisasi Hive telah ditulis. Tampilan masih sementara.

---

# Commit 3 — GetX Controller, Binding, dan Routing

## 1. Isi `lib/controllers/servis_controller.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

import '../models/servis_model.dart';
import '../routes/app_routes.dart';
import '../services/hive_service.dart';

class ServisController extends GetxController {
  final HiveService _service = HiveService();

  final servisList = <Servis>[].obs;
  final keyword = ''.obs;
  final selectedJenis = 'Motor'.obs;

  final List<String> daftarJenis = const [
    'Motor',
    'Mobil',
  ];

  final formKey = GlobalKey<FormState>();
  final platNomorC = TextEditingController();
  final namaPemilikC = TextEditingController();
  final keluhanC = TextEditingController();
  final biayaC = TextEditingController();

  String? editingId;

  bool get isEdit => editingId != null;

  List<Servis> get filteredList {
    final q = keyword.value.toLowerCase().trim();
    if (q.isEmpty) return servisList;

    return servisList.where((servis) {
      return servis.platNomor.toLowerCase().contains(q) ||
          servis.namaPemilik.toLowerCase().contains(q);
    }).toList();
  }

  @override
  void onInit() {
    super.onInit();
    loadData();
  }

  void loadData() {
    final data = _service.getAll()
      ..sort((a, b) => a.namaPemilik
          .toLowerCase()
          .compareTo(b.namaPemilik.toLowerCase()));
    servisList.assignAll(data);
  }

  void openAddForm() {
    editingId = null;
    platNomorC.clear();
    namaPemilikC.clear();
    keluhanC.clear();
    biayaC.clear();
    selectedJenis.value = daftarJenis.first;
    Get.toNamed(AppRoutes.form);
  }

  void openEditForm(Servis servis) {
    editingId = servis.id;
    platNomorC.text = servis.platNomor;
    namaPemilikC.text = servis.namaPemilik;
    keluhanC.text = servis.keluhan;
    biayaC.text = servis.biaya.toStringAsFixed(0);
    selectedJenis.value = servis.jenisKendaraan;
    Get.toNamed(AppRoutes.form);
  }

  Future<void> save() async {
    if (!formKey.currentState!.validate()) return;

    final platNomor = platNomorC.text.trim().toUpperCase();
    final biaya = double.parse(biayaC.text.trim());
    final servisLama = isEdit
        ? servisList.firstWhere((s) => s.id == editingId)
        : null;

    final servis = Servis(
      id: editingId ?? DateTime.now().microsecondsSinceEpoch.toString(),
      platNomor: platNomor,
      namaPemilik: namaPemilikC.text.trim(),
      jenisKendaraan: selectedJenis.value,
      keluhan: keluhanC.text.trim(),
      biaya: biaya,
      selesai: servisLama?.selesai ?? false,
    );

    if (isEdit) {
      await _service.update(servis);
    } else {
      await _service.create(servis);
    }

    final pesan = isEdit
        ? 'Data servis berhasil diperbarui'
        : 'Data servis berhasil ditambahkan';

    loadData();
    tutup();
    Get.snackbar('Sukses', pesan, snackPosition: SnackPosition.BOTTOM);
  }

  Future<void> confirmDelete(Servis servis) async {
    final yakin = await Get.dialog<bool>(
      AlertDialog(
        title: const Text('Hapus Data'),
        content: Text('Yakin ingin menghapus servis ${servis.platNomor}?'),
        actions: [
          TextButton(
            onPressed: () => tutup(false),
            child: const Text('Batal'),
          ),
          FilledButton(
            onPressed: () => tutup(true),
            child: const Text('Hapus'),
          ),
        ],
      ),
    );

    if (yakin == true) {
      await _service.delete(servis.id);
      loadData();
      Get.snackbar(
        'Terhapus',
        'Data servis ${servis.platNomor} berhasil dihapus',
        snackPosition: SnackPosition.BOTTOM,
      );
    }
  }

  void tutup<T>([T? result]) {
    Get.key.currentState?.pop<T>(result);
  }

  String? validatePlatNomor(String? value) {
    if (value == null || value.trim().isEmpty) {
      return 'Plat nomor wajib diisi';
    }
    return null;
  }

  String? validateNamaPemilik(String? value) {
    if (value == null || value.trim().isEmpty) {
      return 'Nama pemilik wajib diisi';
    }
    return null;
  }

  String? validateKeluhan(String? value) {
    if (value == null || value.trim().isEmpty) {
      return 'Keluhan wajib diisi';
    }
    return null;
  }

  String? validateBiaya(String? value) {
    if (value == null || value.trim().isEmpty) {
      return 'Biaya wajib diisi';
    }

    final biaya = double.tryParse(value.trim());
    if (biaya == null) {
      return 'Biaya harus berupa angka';
    }
    if (!biaya.isFinite || biaya < 0) {
      return 'Biaya minimal 0';
    }
    return null;
  }

  @override
  void onClose() {
    platNomorC.dispose();
    namaPemilikC.dispose();
    keluhanC.dispose();
    biayaC.dispose();
    super.onClose();
  }
}
```

Controller mengelola validasi, CRUD, pencarian dan navigasi. Variabel `.obs` dipantau oleh `Obx` yang dibuat pada tahap selanjutnya.

## 2. Isi `lib/bindings/servis_binding.dart`

```dart
import 'package:get/get.dart';

import '../controllers/servis_controller.dart';

class ServisBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut<ServisController>(() => ServisController());
  }
}
```

## 3. Isi `lib/routes/app_routes.dart`

```dart
abstract class AppRoutes {
  static const home = '/';
  static const form = '/form';
}
```

## 4. Buat View sementara

Isi `lib/views/servis_list_view.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

import '../controllers/servis_controller.dart';

class ServisListView extends GetView<ServisController> {
  const ServisListView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Antrian Servis Bengkel'),
        centerTitle: true,
      ),
      body: const Center(child: Text('Halaman Daftar Servis')),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: controller.openAddForm,
        icon: const Icon(Icons.add),
        label: const Text('Tambah'),
      ),
    );
  }
}
```

Isi `lib/views/servis_form_view.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

import '../controllers/servis_controller.dart';

class ServisFormView extends GetView<ServisController> {
  const ServisFormView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(controller.isEdit ? 'Edit Servis' : 'Tambah Servis'),
      ),
      body: const Center(child: Text('Halaman Form Servis')),
    );
  }
}
```

## 5. Isi `lib/routes/app_pages.dart`

```dart
import 'package:get/get.dart';

import '../bindings/servis_binding.dart';
import '../views/servis_form_view.dart';
import '../views/servis_list_view.dart';
import 'app_routes.dart';

class AppPages {
  static final pages = [
    GetPage(
      name: AppRoutes.home,
      page: () => const ServisListView(),
      binding: ServisBinding(),
    ),
    GetPage(
      name: AppRoutes.form,
      page: () => const ServisFormView(),
    ),
  ];
}
```

## 6. Ganti `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:hive_flutter/hive_flutter.dart';

import 'models/servis_adapter.dart';
import 'models/servis_model.dart';
import 'routes/app_pages.dart';
import 'routes/app_routes.dart';
import 'services/hive_service.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  Hive.registerAdapter(ServisAdapter());
  await Hive.openBox<Servis>(HiveService.boxName);
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: 'Antrian Servis Bengkel',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorSchemeSeed: Colors.indigo,
        useMaterial3: true,
      ),
      initialRoute: AppRoutes.home,
      getPages: AppPages.pages,
    );
  }
}
```

## 7. Verifikasi dan commit

```bash
dart format lib
flutter analyze
flutter run
git add lib/main.dart lib/controllers/ lib/bindings/ lib/routes/ lib/views/
git commit -m "feat: add GetX controller and CRUD operations"
```

Uji tombol **Tambah** dan navigasi kembali. Form dan daftar masih placeholder, bukan bug.

---

# Commit 4 — View CRUD Lengkap

## 1. Tambahkan package format Rupiah

```bash
flutter pub add intl
```

## 2. Ganti `lib/views/servis_list_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:intl/intl.dart';

import '../controllers/servis_controller.dart';
import '../models/servis_model.dart';

class ServisListView extends GetView<ServisController> {
  const ServisListView({super.key});

  String formatRupiah(double nilai) {
    return NumberFormat.currency(
      locale: 'id_ID',
      symbol: 'Rp ',
      decimalDigits: 0,
    ).format(nilai);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Antrian Servis Bengkel'),
        centerTitle: true,
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: TextField(
              onChanged: (value) => controller.keyword.value = value,
              decoration: const InputDecoration(
                hintText: 'Cari plat nomor atau pemilik...',
                prefixIcon: Icon(Icons.search),
                border: OutlineInputBorder(),
              ),
            ),
          ),
          Expanded(
            child: Obx(() {
              final data = controller.filteredList;
              if (data.isEmpty) {
                return const Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(Icons.build_outlined, size: 64, color: Colors.grey),
                      SizedBox(height: 12),
                      Text(
                        'Belum ada data servis',
                        style: TextStyle(color: Colors.grey),
                      ),
                    ],
                  ),
                );
              }

              return ListView.builder(
                padding: const EdgeInsets.only(bottom: 80),
                itemCount: data.length,
                itemBuilder: (context, index) => _buildServisCard(data[index]),
              );
            }),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: controller.openAddForm,
        icon: const Icon(Icons.add),
        label: const Text('Tambah Servis'),
      ),
    );
  }

  Widget _buildServisCard(Servis servis) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 6),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                CircleAvatar(
                  child: Icon(
                    servis.jenisKendaraan == 'Motor'
                        ? Icons.two_wheeler
                        : Icons.directions_car,
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        servis.platNomor,
                        style: const TextStyle(
                          fontSize: 16,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(servis.namaPemilik),
                    ],
                  ),
                ),
                PopupMenuButton<String>(
                  onSelected: (value) {
                    if (value == 'edit') {
                      controller.openEditForm(servis);
                    } else if (value == 'hapus') {
                      controller.confirmDelete(servis);
                    }
                  },
                  itemBuilder: (context) => [
                    const PopupMenuItem(
                      value: 'edit',
                      child: Row(
                        children: [
                          Icon(Icons.edit_outlined),
                          SizedBox(width: 8),
                          Text('Edit'),
                        ],
                      ),
                    ),
                    const PopupMenuItem(
                      value: 'hapus',
                      child: Row(
                        children: [
                          Icon(Icons.delete_outline),
                          SizedBox(width: 8),
                          Text('Hapus'),
                        ],
                      ),
                    ),
                  ],
                ),
              ],
            ),
            const Divider(height: 24),
            Text('Jenis Kendaraan: ${servis.jenisKendaraan}'),
            const SizedBox(height: 6),
            Text('Keluhan: ${servis.keluhan}'),
            const SizedBox(height: 12),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  formatRupiah(servis.biaya),
                  style: const TextStyle(
                    fontSize: 16,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Chip(
                  label: Text(servis.selesai ? 'Selesai' : 'Proses'),
                  backgroundColor: servis.selesai
                      ? Colors.green.shade100
                      : Colors.orange.shade100,
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

## 3. Ganti `lib/views/servis_form_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:get/get.dart';

import '../controllers/servis_controller.dart';

class ServisFormView extends GetView<ServisController> {
  const ServisFormView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(controller.isEdit ? 'Edit Servis' : 'Tambah Servis'),
      ),
      body: Form(
        key: controller.formKey,
        autovalidateMode: AutovalidateMode.onUserInteraction,
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: [
            TextFormField(
              controller: controller.platNomorC,
              textCapitalization: TextCapitalization.characters,
              inputFormatters: [UpperCaseTextFormatter()],
              validator: controller.validatePlatNomor,
              decoration: const InputDecoration(
                labelText: 'Plat Nomor',
                hintText: 'AB 1234 CD',
                prefixIcon: Icon(Icons.pin_outlined),
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: controller.namaPemilikC,
              textCapitalization: TextCapitalization.words,
              validator: controller.validateNamaPemilik,
              decoration: const InputDecoration(
                labelText: 'Nama Pemilik',
                hintText: 'Masukkan nama pemilik',
                prefixIcon: Icon(Icons.person_outline),
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 16),
            DropdownButtonFormField<String>(
              initialValue: controller.selectedJenis.value,
              items: controller.daftarJenis.map((jenis) {
                return DropdownMenuItem(value: jenis, child: Text(jenis));
              }).toList(),
              onChanged: (value) {
                if (value != null) controller.selectedJenis.value = value;
              },
              decoration: const InputDecoration(
                labelText: 'Jenis Kendaraan',
                prefixIcon: Icon(Icons.directions_car_outlined),
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: controller.keluhanC,
              maxLines: 3,
              validator: controller.validateKeluhan,
              decoration: const InputDecoration(
                labelText: 'Keluhan',
                hintText: 'Masukkan keluhan kendaraan',
                prefixIcon: Icon(Icons.build_outlined),
                border: OutlineInputBorder(),
                alignLabelWithHint: true,
              ),
            ),
            const SizedBox(height: 16),
            TextFormField(
              controller: controller.biayaC,
              keyboardType: const TextInputType.numberWithOptions(decimal: true),
              inputFormatters: [
                FilteringTextInputFormatter.allow(RegExp(r'[0-9.]')),
              ],
              validator: controller.validateBiaya,
              decoration: const InputDecoration(
                labelText: 'Biaya Servis',
                hintText: '150000',
                prefixIcon: Icon(Icons.payments_outlined),
                prefixText: 'Rp ',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: controller.save,
              icon: const Icon(Icons.save_outlined),
              label: Text(controller.isEdit ? 'Perbarui' : 'Simpan'),
              style: FilledButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class UpperCaseTextFormatter extends TextInputFormatter {
  @override
  TextEditingValue formatEditUpdate(
    TextEditingValue oldValue,
    TextEditingValue newValue,
  ) {
    return newValue.copyWith(text: newValue.text.toUpperCase());
  }
}
```

Jika menggunakan Flutter versi yang tidak mengenali `initialValue` pada `DropdownButtonFormField`, sesuaikan dengan API versi Flutter yang terpasang. Modul memakai Flutter 3.x versi baru.

## 4. Uji CRUD dan commit

```bash
dart format lib
flutter analyze
flutter run
```

Skenario manual: tambah servis, coba form kosong, cari plat atau pemilik, edit biaya, hapus dengan Batal dan Hapus, lalu buka ulang aplikasi untuk mengecek data masih tersimpan.

```bash
git add lib/views/servis_list_view.dart lib/views/servis_form_view.dart pubspec.yaml pubspec.lock
git commit -m "feat: build service list and form views"
```

**Selesai Commit 4:** form dan daftar CRUD dapat digunakan. Fitur status khusus belum ditambahkan.

---

# Commit 5 — Filter Status, Tandai Selesai, dan Total Pendapatan

## 1. Ubah `lib/controllers/servis_controller.dart`

Di dalam class `ServisController`, setelah deklarasi `selectedJenis`, tambahkan:

```dart
final selectedStatus = 'Proses'.obs;

final List<String> daftarStatus = const [
  'Proses',
  'Selesai',
];
```

Ganti getter `filteredList` lama dengan:

```dart
List<Servis> get filteredList {
  final q = keyword.value.toLowerCase().trim();
  final status = selectedStatus.value;

  return servisList.where((servis) {
    final cocokPencarian =
        servis.platNomor.toLowerCase().contains(q) ||
        servis.namaPemilik.toLowerCase().contains(q);

    final cocokStatus = status == 'Proses'
        ? !servis.selesai
        : servis.selesai;

    return cocokPencarian && cocokStatus;
  }).toList();
}
```

Tambahkan getter `totalPendapatan`:

```dart
double get totalPendapatan {
  return servisList
      .where((servis) => servis.selesai)
      .fold<double>(0, (total, servis) => total + servis.biaya);
}
```

Tambahkan method `tandaiSelesai` di dalam class:

```dart
Future<void> tandaiSelesai(Servis servis) async {
  if (servis.selesai) return;

  final servisBaru = servis.copyWith(selesai: true);
  await _service.update(servisBaru);
  loadData();

  Get.snackbar(
    'Sukses',
    'Servis ${servis.platNomor} telah selesai',
    snackPosition: SnackPosition.BOTTOM,
  );
}
```

Jangan menggandakan getter `filteredList`: yang lama **diganti**, bukan ditambahkan sebagai getter kedua.

## 2. Ganti `lib/views/servis_list_view.dart`

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:intl/intl.dart';

import '../controllers/servis_controller.dart';
import '../models/servis_model.dart';

class ServisListView extends GetView<ServisController> {
  const ServisListView({super.key});

  String formatRupiah(double nilai) {
    return NumberFormat.currency(
      locale: 'id_ID',
      symbol: 'Rp ',
      decimalDigits: 0,
    ).format(nilai);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Antrian Servis Bengkel'),
        centerTitle: true,
      ),
      body: Column(
        children: [
          Obx(() {
            final total = controller.totalPendapatan;
            return Card(
              margin: const EdgeInsets.fromLTRB(16, 16, 16, 8),
              color: Theme.of(context).colorScheme.primaryContainer,
              child: Padding(
                padding: const EdgeInsets.all(20),
                child: Row(
                  children: [
                    const Icon(Icons.account_balance_wallet_outlined, size: 36),
                    const SizedBox(width: 16),
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          const Text(
                            'Total Pendapatan',
                            style: TextStyle(fontSize: 14),
                          ),
                          const SizedBox(height: 4),
                          Text(
                            formatRupiah(total),
                            style: const TextStyle(
                              fontSize: 24,
                              fontWeight: FontWeight.bold,
                            ),
                          ),
                          const Text(
                            'Dari servis yang telah selesai',
                            style: TextStyle(fontSize: 12),
                          ),
                        ],
                      ),
                    ),
                  ],
                ),
              ),
            );
          }),
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 8, 16, 12),
            child: TextField(
              onChanged: (value) => controller.keyword.value = value,
              decoration: const InputDecoration(
                hintText: 'Cari plat nomor atau pemilik...',
                prefixIcon: Icon(Icons.search),
                border: OutlineInputBorder(),
              ),
            ),
          ),
          Obx(() {
            return Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Row(
                children: controller.daftarStatus.map((status) {
                  final aktif = controller.selectedStatus.value == status;
                  return Expanded(
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 4),
                      child: ChoiceChip(
                        label: SizedBox(
                          width: double.infinity,
                          child: Text(status, textAlign: TextAlign.center),
                        ),
                        selected: aktif,
                        showCheckmark: false,
                        onSelected: (_) {
                          controller.selectedStatus.value = status;
                        },
                      ),
                    ),
                  );
                }).toList(),
              ),
            );
          }),
          const SizedBox(height: 8),
          Expanded(
            child: Obx(() {
              final data = controller.filteredList;
              if (data.isEmpty) {
                return Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const Icon(
                        Icons.build_outlined,
                        size: 64,
                        color: Colors.grey,
                      ),
                      const SizedBox(height: 12),
                      Text(
                        controller.keyword.value.isNotEmpty
                            ? 'Data tidak ditemukan'
                            : 'Belum ada servis ${controller.selectedStatus.value.toLowerCase()}',
                        style: const TextStyle(color: Colors.grey),
                      ),
                    ],
                  ),
                );
              }

              return ListView.builder(
                padding: const EdgeInsets.only(bottom: 80),
                itemCount: data.length,
                itemBuilder: (context, index) => _buildServisCard(data[index]),
              );
            }),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: controller.openAddForm,
        icon: const Icon(Icons.add),
        label: const Text('Tambah Servis'),
      ),
    );
  }

  Widget _buildServisCard(Servis servis) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 6),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                CircleAvatar(
                  child: Icon(
                    servis.jenisKendaraan == 'Motor'
                        ? Icons.two_wheeler
                        : Icons.directions_car,
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        servis.platNomor,
                        style: const TextStyle(
                          fontSize: 16,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(servis.namaPemilik),
                    ],
                  ),
                ),
                PopupMenuButton<String>(
                  onSelected: (value) {
                    if (value == 'edit') {
                      controller.openEditForm(servis);
                    } else if (value == 'hapus') {
                      controller.confirmDelete(servis);
                    }
                  },
                  itemBuilder: (context) => [
                    const PopupMenuItem(
                      value: 'edit',
                      child: Row(
                        children: [
                          Icon(Icons.edit_outlined),
                          SizedBox(width: 8),
                          Text('Edit'),
                        ],
                      ),
                    ),
                    const PopupMenuItem(
                      value: 'hapus',
                      child: Row(
                        children: [
                          Icon(Icons.delete_outline),
                          SizedBox(width: 8),
                          Text('Hapus'),
                        ],
                      ),
                    ),
                  ],
                ),
              ],
            ),
            const Divider(height: 24),
            Text('Jenis Kendaraan: ${servis.jenisKendaraan}'),
            const SizedBox(height: 6),
            Text('Keluhan: ${servis.keluhan}'),
            const SizedBox(height: 12),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  formatRupiah(servis.biaya),
                  style: const TextStyle(
                    fontSize: 16,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Chip(
                  label: Text(servis.selesai ? 'Selesai' : 'Proses'),
                  backgroundColor: servis.selesai
                      ? Colors.green.shade100
                      : Colors.orange.shade100,
                ),
              ],
            ),
            if (!servis.selesai) ...[
              const SizedBox(height: 12),
              SizedBox(
                width: double.infinity,
                child: OutlinedButton.icon(
                  onPressed: () => controller.tandaiSelesai(servis),
                  icon: const Icon(Icons.check_circle_outline),
                  label: const Text('Tandai Selesai'),
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

## 3. Uji fitur dan commit

```bash
dart format lib
flutter analyze
flutter run
```

Pengujian: data baru berada di Proses; Tandai Selesai memindahkannya ke Selesai; total pendapatan hanya menjumlahkan biaya servis selesai; edit/hapus servis selesai memperbarui pendapatan.

```bash
git add lib/controllers/servis_controller.dart lib/views/servis_list_view.dart
git commit -m "feat: add service status filters and revenue summary"
```

**Selesai Commit 5:** seluruh tiga fitur khusus Paket 4 sudah ditambahkan.

---

# Commit 6 — Unit Test, Dokumentasi, dan Pengumpulan

## 1. Hapus test template Flutter lama

File `test/widget_test.dart` bawaan Flutter menguji aplikasi counter dan tidak cocok untuk aplikasi bengkel. Hapus agar `flutter test` tidak gagal akibat test lama.

```bash
rm -f test/widget_test.dart
```

## 2. Buat `test/hive_service_test.dart`

```dart
import 'dart:io';

import 'package:flutter_test/flutter_test.dart';
import 'package:hive/hive.dart';

import 'package:antrian_servis_bengkel/models/servis_adapter.dart';
import 'package:antrian_servis_bengkel/models/servis_model.dart';
import 'package:antrian_servis_bengkel/services/hive_service.dart';

void main() {
  late Directory dir;
  final service = HiveService();

  setUpAll(() async {
    dir = await Directory.systemTemp.createTemp('hive_test');
    Hive.init(dir.path);
    Hive.registerAdapter(ServisAdapter());
    await Hive.openBox<Servis>(HiveService.boxName);
  });

  setUp(() async {
    await Hive.box<Servis>(HiveService.boxName).clear();
  });

  tearDownAll(() async {
    await Hive.close();
    await dir.delete(recursive: true);
  });

  final servis = Servis(
    id: '1',
    platNomor: 'AB 1234 CD',
    namaPemilik: 'Budi Santoso',
    jenisKendaraan: 'Motor',
    keluhan: 'Ganti oli',
    biaya: 150000,
  );

  test('CREATE - Menambahkan data servis', () async {
    await service.create(servis);
    final data = service.getAll();
    expect(data.length, 1);
    expect(data.first.platNomor, 'AB 1234 CD');
    expect(data.first.namaPemilik, 'Budi Santoso');
    expect(data.first.selesai, false);
  });

  test('READ - Membaca data servis', () async {
    await service.create(servis);
    final data = service.getAll();
    expect(data, isNotEmpty);
    expect(data.first.jenisKendaraan, 'Motor');
    expect(data.first.keluhan, 'Ganti oli');
    expect(data.first.biaya, 150000);
  });

  test('UPDATE - Mengubah data servis', () async {
    await service.create(servis);
    final servisBaru = servis.copyWith(
      keluhan: 'Ganti oli dan servis rem',
      biaya: 250000,
    );
    await service.update(servisBaru);
    final data = service.getAll();
    expect(data.length, 1);
    expect(data.first.keluhan, 'Ganti oli dan servis rem');
    expect(data.first.biaya, 250000);
  });

  test('UPDATE - Mengubah status menjadi selesai', () async {
    await service.create(servis);
    final servisSelesai = servis.copyWith(selesai: true);
    await service.update(servisSelesai);
    final data = service.getAll();
    expect(data.first.selesai, true);
  });

  test('DELETE - Menghapus data servis', () async {
    await service.create(servis);
    await service.delete(servis.id);
    final data = service.getAll();
    expect(data, isEmpty);
  });

  test('PERSISTENSI - Data tetap ada setelah box dibuka lagi', () async {
    await service.create(servis);
    await Hive.box<Servis>(HiveService.boxName).close();
    await Hive.openBox<Servis>(HiveService.boxName);
    final data = service.getAll();
    expect(data.length, 1);
    expect(data.first.platNomor, 'AB 1234 CD');
  });
}
```

Kode test di atas menguji enam skenario: Create, Read, Update, perubahan status, Delete, dan persistensi. Jika nama proyek pada `pubspec.yaml` berbeda, sesuaikan bagian `package:antrian_servis_bengkel/...`.

## 3. Jalankan pemeriksaan

```bash
dart format lib test
flutter analyze
flutter test
```

Target hasil ketika semua berjalan benar:

```text
No issues found!
+6: All tests passed!
```

Jika `widget_test.dart` masih ada dan menghasilkan error `Box not found` atau `Counter increments smoke test`, pastikan file template tersebut sudah dihapus lalu jalankan `flutter test` kembali.

## 4. Ambil screenshot asli aplikasi

```bash
mkdir -p screenshots
```

Simpan screenshot sebagai:

```text
screenshots/
├── daftar-servis.png
├── tambah-servis.png
├── validasi-form.png
├── edit-servis.png
├── status-proses.png
├── status-selesai.png
└── total-pendapatan.png
```

Gunakan screenshot dari emulator/HP sendiri; jangan menganggap mockup sebagai hasil pengujian aplikasi.

## 5. Isi `README.md` proyek

Ganti identitas dan tautan placeholder dengan data sebenarnya.

````markdown
# Antrian Servis Bengkel

## Identitas

- Nama: [Nama Lengkap]
- NIM: [NIM]
- Paket: 4 — Antrian Servis Bengkel
- Mata kuliah: Pemrograman Mobile

## Deskripsi

Aplikasi mobile untuk mencatat dan mengelola antrian servis motor/mobil. Data servis disimpan lokal menggunakan Hive sehingga tetap tersedia setelah aplikasi ditutup.

## Teknologi

- Flutter dan Dart
- Hive dan hive_flutter
- GetX
- MVC
- Material 3
- Intl

## Fitur

- Tambah, tampil, cari, ubah dan hapus servis
- Validasi plat nomor, pemilik, keluhan, dan biaya
- Dropdown jenis kendaraan Motor/Mobil
- Filter Proses dan Selesai
- Tombol Tandai Selesai
- Total pendapatan dari servis selesai
- Penyimpanan lokal persisten

## Struktur Proyek

```text
lib/
├── main.dart
├── models/
│   ├── servis_model.dart
│   └── servis_adapter.dart
├── services/
│   └── hive_service.dart
├── controllers/
│   └── servis_controller.dart
├── views/
│   ├── servis_list_view.dart
│   └── servis_form_view.dart
├── bindings/
│   └── servis_binding.dart
└── routes/
    ├── app_routes.dart
    └── app_pages.dart
```

## Menjalankan Aplikasi

```bash
flutter pub get
flutter run
```

## Pengujian

```bash
flutter analyze
flutter test
```

Enam test service mencakup Create, Read, Update, perubahan status, Delete, serta persistensi data.

## Screenshot

![Daftar Servis](screenshots/daftar-servis.png)
![Tambah Servis](screenshots/tambah-servis.png)
![Validasi Form](screenshots/validasi-form.png)
![Edit Servis](screenshots/edit-servis.png)
![Status Proses](screenshots/status-proses.png)
![Status Selesai](screenshots/status-selesai.png)
![Total Pendapatan](screenshots/total-pendapatan.png)

## Video Presentasi

[Video penjelasan aplikasi](GANTI_DENGAN_URL_VIDEO_YOUTUBE)
````

## 6. Commit keenam

Setelah test lulus, screenshot dan README selesai:

```bash
git status
git add -A
git commit -m "test: add CRUD tests and project documentation"
git log --oneline
```

**Penting:** Jika video diunggah setelah commit keenam, penambahan link YouTube pada README memerlukan commit dokumentasi tambahan, atau masukkan link sebelum commit keenam. Jumlah lebih dari enam commit diperbolehkan, karena ketentuan hanya **minimal lima**.

---

# Membuat Repository Private dan Push ke GitHub

1. Buat repository baru di GitHub dengan nama `cpmk03-<NIM>-paket4`, misalnya `cpmk03-2201010124-paket4`.
2. Pilih **Private**.
3. Jangan centang pembuatan README baru karena sudah ada di proyek lokal.
4. Hubungkan repo lokal dan push.

```bash
git branch -M main
git remote add origin https://github.com/USERNAME/cpmk03-NIM-paket4.git
git push -u origin main
```

Jika `origin` sudah ada, periksa dulu dengan `git remote -v` dan jangan menjalankan `git remote add origin` lagi.

Tambahkan username **`mohammadirham37`** melalui **Settings → Collaborators → Add people**. Repository harus dapat diakses dosen.

## Riwayat commit yang direncanakan

| Urutan | Commit message |
|---|---|
| 1 | `chore: initialize Flutter project with MVC structure` |
| 2 | `feat: implement service model and Hive database` |
| 3 | `feat: add GetX controller and CRUD operations` |
| 4 | `feat: build service list and form views` |
| 5 | `feat: add service status filters and revenue summary` |
| 6 | `test: add CRUD tests and project documentation` |

## Video ujian dan pengumpulan

- Durasi video minimal **5 menit**; gunakan suara sendiri.
- Judul YouTube: `CPMK 03 - Paket 4 - Nama Lengkap - NIM`.
- Visibilitas **Unlisted** atau **Public**, bukan Private.
- Tampilkan struktur MVC, Model/Adapter/Service, Controller/GetX, Binding dan Route, View, CRUD, validasi, fitur khusus, data persisten, dan hasil `flutter test`.
- Kirim **link repository private** dan **link video YouTube** ke LMS.

## Checklist akhir

- [ ] Aplikasi bisa dijalankan tanpa error
- [ ] CRUD dan pencarian berjalan
- [ ] Semua validasi berjalan
- [ ] Filter Proses/Selesai berjalan
- [ ] Tombol Tandai Selesai berjalan
- [ ] Total pendapatan hanya menghitung servis Selesai
- [ ] Data tetap ada setelah aplikasi ditutup
- [ ] `flutter analyze` berhasil
- [ ] Enam unit test lulus
- [ ] README dan screenshot asli lengkap
- [ ] Commit dilakukan bertahap
- [ ] Repository private sudah di-push
- [ ] `mohammadirham37` diundang sebagai collaborator
- [ ] Video YouTube minimal 5 menit bisa diputar
- [ ] Link GitHub dan video sudah dikumpulkan ke LMS

---

*Panduan ini merekonstruksi tahap Commit 1–6 yang telah dibahas. Hasil menjalankan aplikasi dan test harus diverifikasi di perangkat pengguna sendiri.*
