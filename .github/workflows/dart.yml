import 'dart:io';

import 'package:flutter/material.dart';
import 'package:geolocator/geolocator.dart';
import 'package:image_picker/image_picker.dart';

void main() {
  runApp(const HoranMunicipalityApp());
}

class HoranMunicipalityApp extends StatelessWidget {
  const HoranMunicipalityApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'بلدية سهل حوران',
      theme: ThemeData(
        brightness: Brightness.dark,
        scaffoldBackgroundColor: Colors.black,
        primaryColor: const Color(0xFFD4AF37),
      ),
      home: const ReportScreen(),
    );
  }
}

class ReportScreen extends StatefulWidget {
  const ReportScreen({super.key});

  @override
  State<ReportScreen> createState() => _ReportScreenState();
}

class _ReportScreenState extends State<ReportScreen> {
  final TextEditingController descriptionController =
      TextEditingController();

  String? selectedArea;
  String? selectedCategory;

  double? latitude;
  double? longitude;

  File? imageFile;

  final List<String> areas = [
    "الطرة",
    "الشجرة",
    "الذنيبة",
    "عمراوة",
  ];

  final List<String> categories = [
    "أعطال إنارة",
    "حفر طرق",
    "تراكم نفايات",
    "تصريف مياه",
    "حدائق ومرافق",
  ];

  @override
  void initState() {
    super.initState();
    getLocation();
  }

  Future<void> getLocation() async {
    bool serviceEnabled =
        await Geolocator.isLocationServiceEnabled();

    if (!serviceEnabled) {
      return;
    }

    LocationPermission permission =
        await Geolocator.checkPermission();

    if (permission == LocationPermission.denied) {
      permission =
          await Geolocator.requestPermission();
    }

    Position position =
        await Geolocator.getCurrentPosition(
      desiredAccuracy: LocationAccuracy.high,
    );

    setState(() {
      latitude = position.latitude;
      longitude = position.longitude;
    });
  }

  Future<void> takePhoto() async {
    final picker = ImagePicker();

    final XFile? image =
        await picker.pickImage(
      source: ImageSource.camera,
      imageQuality: 80,
    );

    if (image != null) {
      setState(() {
        imageFile = File(image.path);
      });
    }
  }

  void sendReport() {
    if (selectedArea == null ||
        selectedCategory == null ||
        descriptionController.text.isEmpty ||
        imageFile == null ||
        latitude == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content:
              Text("يرجى إدخال جميع البيانات"),
        ),
      );
      return;
    }

    print({
      "municipality": "بلدية سهل حوران",
      "area": selectedArea,
      "category": selectedCategory,
      "description":
          descriptionController.text,
      "lat": latitude,
      "lng": longitude,
      "image": imageFile!.path,
    });

    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        backgroundColor: Color(0xFFD4AF37),
        content: Text(
          "تم إرسال البلاغ بنجاح",
          style: TextStyle(color: Colors.black),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    const gold = Color(0xFFD4AF37);

    return Scaffold(
      appBar: AppBar(
        centerTitle: true,
        backgroundColor: Colors.black,
        elevation: 0,
        title: const Text(
          "بلدية سهل حوران",
          style: TextStyle(color: gold),
        ),
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [

            const SizedBox(height: 10),

            const Text(
              "حوران الجنة",
              style: TextStyle(
                color: gold,
                fontSize: 28,
                fontWeight: FontWeight.bold,
              ),
            ),

            const SizedBox(height: 25),

            DropdownButtonFormField<String>(
              value: selectedArea,
              dropdownColor: Colors.black,
              decoration: const InputDecoration(
                labelText: "المنطقة",
                labelStyle:
                    TextStyle(color: gold),
                enabledBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
                focusedBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
              ),
              items: areas
                  .map(
                    (e) => DropdownMenuItem(
                      value: e,
                      child: Text(e),
                    ),
                  )
                  .toList(),
              onChanged: (v) {
                setState(() {
                  selectedArea = v;
                });
              },
            ),

            const SizedBox(height: 15),

            DropdownButtonFormField<String>(
              value: selectedCategory,
              dropdownColor: Colors.black,
              decoration: const InputDecoration(
                labelText: "نوع العطل",
                labelStyle:
                    TextStyle(color: gold),
                enabledBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
                focusedBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
              ),
              items: categories
                  .map(
                    (e) => DropdownMenuItem(
                      value: e,
                      child: Text(e),
                    ),
                  )
                  .toList(),
              onChanged: (v) {
                setState(() {
                  selectedCategory = v;
                });
              },
            ),

            const SizedBox(height: 15),

            TextField(
              controller: descriptionController,
              maxLines: 5,
              decoration: const InputDecoration(
                labelText: "وصف المشكلة",
                labelStyle:
                    TextStyle(color: gold),
                enabledBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
                focusedBorder:
                    OutlineInputBorder(
                  borderSide:
                      BorderSide(color: gold),
                ),
              ),
            ),

            const SizedBox(height: 20),

            Container(
              padding:
                  const EdgeInsets.all(15),
              width: double.infinity,
              decoration: BoxDecoration(
                border:
                    Border.all(color: gold),
                borderRadius:
                    BorderRadius.circular(12),
              ),
              child: Column(
                children: [
                  const Text(
                    "الموقع الحالي GPS",
                    style:
                        TextStyle(color: gold),
                  ),
                  const SizedBox(height: 10),
                  Text(
                    "LAT : ${latitude ?? "جاري التحميل"}",
                  ),
                  Text(
                    "LNG : ${longitude ?? "جاري التحميل"}",
                  ),
                ],
              ),
            ),

            const SizedBox(height: 20),

            ElevatedButton.icon(
              onPressed: takePhoto,
              style: ElevatedButton.styleFrom(
                backgroundColor: gold,
                foregroundColor:
                    Colors.black,
              ),
              icon:
                  const Icon(Icons.camera_alt),
              label:
                  const Text("التقاط صورة"),
            ),

            const SizedBox(height: 20),

            if (imageFile != null)
              ClipRRect(
                borderRadius:
                    BorderRadius.circular(12),
                child: Image.file(
                  imageFile!,
                  height: 220,
                  width: double.infinity,
                  fit: BoxFit.cover,
                ),
              ),

            const SizedBox(height: 30),

            SizedBox(
              width: double.infinity,
              height: 55,
              child: ElevatedButton(
                onPressed: sendReport,
                style:
                    ElevatedButton.styleFrom(
                  backgroundColor: gold,
                  foregroundColor:
                      Colors.black,
                ),
                child: const Text(
                  "إرسال البلاغ",
                  style: TextStyle(
                    fontSize: 18,
                    fontWeight:
                        FontWeight.bold,
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
