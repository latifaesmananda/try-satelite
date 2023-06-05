# try-satelite

TIF adalah format gambar yang menyimpan nilai statistik di setiap piksel gambar. Meskipun setelah masking awan, nilai statistik tidak akan terganggu karna hanya berganti menjadi 0 saja.

### 1. Sentinel-2 Halim, without preprocessing. Only median
<img width="514" alt="image" src="https://github.com/latifaesmananda/try-satelite/assets/121326117/a57dd022-af77-44b0-8988-37aabf9f47a3">

```
// Pusatkan titik observasi di Peta
Map.centerObject(halim)

// Pilih citra satelit untuk melihat permukaan bumi (satelit copernicus)
var dataset = ee.ImageCollection('COPERNICUS/S2_HARMONIZED')
                  .filterDate('2019-03-01', '2019-03-31');
                  
// Reduce nilai (harus ditetapkan 1 nilai)
// sentinel-2 punya resolusi temporal 10 hari, karna konstelasi jadi 5 hari
// berarti kalo sebulan, dia ada 6 citra
// bagaimana kita mendapatkan nilainya? pake reducer
// dari 6 citra misal ingin diambil 1 nilai rata2 dari keenam citra
var dataset_median = dataset.reduce(ee.Reducer.median())

// Menambahkan layer untuk menampilkan citra satelit
// jika interval warna yg ditampilkan terlalu kecil maka coba diganti
// caranya ke inspector, klik di peta, lihat nilai permasing2 band 
var rgbVis = { //untuk milih band apa saja yang ditampilkan
  min: 1000,
  max: 3000,
  bands: ['B4', 'B3', 'B2'],
};

var rgbVis_median = { //untuk milih band apa saja yang ditampilkan
  min: 1000,
  max: 3000,
  bands: ['B4_median', 'B3_median', 'B2_median'],
};

// Tambah layer citra di peta
Map.addLayer(dataset, rgbVis, 'RGB'); //'RGB' parameter untuk memberi nama pada layer
Map.addLayer(dataset_median, rgbVis_median, 'RGB median');

// Unduh citra satelit (export to Google Drive)
Export.image.toDrive({
  image: dataset_median,
  description: 'halim_maret_2022',// nama file
  folder: 'pengdat-praktek',
  region: halim,
  scale: 10 //resolusi spasial, 1 piksel akan menggambarkan 10 meter
})
```
- coba cek di drive, abistu di download dan coba open di QGIS
- double click layer tif dan ganti bandnya sesuai code, B4 B3 B2

### 2. Sentinel-2 Halim, modif band. Only median + ilangin awan
- double click layer tif dan ganti bandnya (red: SWIR (band 11), green: NIR (band 8), blue: RED (band 4))
- area vegetasi yang hijau
- area terbangun yang merah
- masih ada awan karena sentinel-2 masih pake sensor pasif (bantuan matahari) 
<img width="513" alt="image" src="https://github.com/latifaesmananda/try-satelite/assets/121326117/07ea827b-0cc7-4a3f-9d2d-3f24f5b3b540">

- ilangin awan pake code:

```
// Pusatkan titik observasi di Peta
Map.centerObject(halim)

// Fungsi masking awan
function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}

// Pilih citra satelit untuk melihat permukaan bumi (satelit copernicus)
var dataset = ee.ImageCollection('COPERNICUS/S2_HARMONIZED')
                  .filterDate('2019-03-01', '2019-03-31');
                  
// Reduce nilai (harus ditetapkan 1 nilai)
var dataset_median = dataset.map(maskS2clouds).reduce(ee.Reducer.median());

// Menambahkan layer untuk menampilkan citra satelit
var rgbVis = { //untuk milih band apa saja yang ditampilkan
  min: 1000,
  max: 3000,
  bands: ['B4', 'B3', 'B2'],
};

var rgbVis_median = { //untuk milih band apa saja yang ditampilkan
  min: 0.01,
  max: 0.2,
  bands: ['B4_median', 'B3_median', 'B2_median'],
};

// Tambah layer citra di peta
Map.addLayer(dataset, rgbVis, 'RGB'); //'RGB' parameter untuk memberi nama pada layer
Map.addLayer(dataset_median, rgbVis_median, 'RGB median');

// Unduh citra satelit (export to Google Drive)
// Export.image.toDrive({
//   image: dataset_median,
//   description: 'halim_maret_2022',// nama file
//   folder: 'pengdat-praktek',
//   region: halim,
//   scale: 10 //resolusi spasial, 1 piksel akan menggambarkan 10 meter
// })
```

### 3. Sentinel-2 Halim, ilangin awan advanced (yg hasil mask ditambal pake data citra hari lain, secara statistik dia nanti akan berubah. tapi kekurangannya gabisa akurat di timeseriesnya meskipun visualnya bagus)

### 4. Mendapatkan vegetation index
```
// Pusatkan titik observasi di Peta
Map.centerObject(halim)

// Fungsi masking awan
function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}

// Pilih citra satelit untuk melihat permukaan bumi (satelit copernicus)
var dataset = ee.ImageCollection('COPERNICUS/S2_HARMONIZED')
                  .filterDate('2019-03-01', '2019-03-31');
                  
// Reduce nilai (harus ditetapkan 1 nilai)
var dataset_median = dataset.map(maskS2clouds).reduce(ee.Reducer.median());

// Kalkulasi Indeks
var s2_compsite = dataset_median.addBands(dataset_median.normalizedDifference(['B11_median','B8_median'])).rename('NDBI')
var s2_compsite = s2_compsite.addBands(s2_compsite.normalizedDifference(['B8_median','B4_median'])).rename('NDVI')

// Menambahkan layer untuk menampilkan citra satelit
var rgbVis = { //untuk milih band apa saja yang ditampilkan
  min: 1000,
  max: 3000,
  bands: ['B4', 'B3', 'B2'],
};

var rgbVis_median = { //untuk milih band apa saja yang ditampilkan
  min: 0.01,
  max: 0.2,
  bands: ['B4_median', 'B3_median', 'B2_median'],
};

// Tambah layer citra di peta
Map.addLayer(dataset, rgbVis, 'RGB'); //'RGB' parameter untuk memberi nama pada layer
Map.addLayer(s2_compsite, {min: 0, max: 1, palette:['white','red']}, 'NDBI'); //ini index visualisasinya gagal

// Unduh citra satelit (export to Google Drive)
Export.image.toDrive({
  image: s2_compsite,
  description: 'halim_maret_2022_ndbi_ndvi',// nama file
  folder: 'pengdat-praktek',
  region: halim,
  scale: 10 //resolusi spasial, 1 piksel akan menggambarkan 10 meter
})
```
- coba langsung aja download tifnya, dibuka di QGIS (masi error hua)

### 5. Ekstrak nilai statistik dari nilai yang berhasil diambil
https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_DYNAMICWORLD_V1
itu dari landmark dynamic world, bandnya uda diatur
<img width="409" alt="image" src="https://github.com/latifaesmananda/try-satelite/assets/121326117/b86beb42-8ad8-4c3a-bf94-ea3193b4b6f6">
```
// Construct a collection of corresponding Dynamic World and Sentinel-2 for
// inspection. Filter the DW and S2 collections by region and date.
var START = ee.Date('2021-04-02');
var END = START.advance(1, 'day');

var colFilter = ee.Filter.and(
    ee.Filter.bounds(ee.Geometry.Point(20.6729, 52.4305)),
    ee.Filter.date(START, END));

var dwCol = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1').filter(colFilter);
var s2Col = ee.ImageCollection('COPERNICUS/S2').filter(colFilter);

// Join corresponding DW and S2 images (by system:index).
var DwS2Col = ee.Join.saveFirst('s2_img').apply(dwCol, s2Col,
    ee.Filter.equals({leftField: 'system:index', rightField: 'system:index'}));

// Extract an example DW image and its source S2 image.
var dwImage = ee.Image(DwS2Col.first());
var s2Image = ee.Image(dwImage.get('s2_img'));

// Create a visualization that blends DW class label with probability.
// Define list pairs of DW LULC label and color.
var CLASS_NAMES = [
    'water', 'trees', 'grass', 'flooded_vegetation', 'crops',
    'shrub_and_scrub', 'built', 'bare', 'snow_and_ice'];

var VIS_PALETTE = [
    '419bdf', '397d49', '88b053', '7a87c6', 'e49635', 'dfc35a', 'c4281b',
    'a59b8f', 'b39fe1'];

// Create an RGB image of the label (most likely class) on [0, 1].
var dwRgb = dwImage
    .select('label')
    .visualize({min: 0, max: 8, palette: VIS_PALETTE})
    .divide(255);

// Get the most likely class probability.
var top1Prob = dwImage.select(CLASS_NAMES).reduce(ee.Reducer.max());

// Create a hillshade of the most likely class probability on [0, 1];
var top1ProbHillshade =
    ee.Terrain.hillshade(top1Prob.multiply(100))
    .divide(255);

// Combine the RGB image with the hillshade.
var dwRgbHillshade = dwRgb.multiply(top1ProbHillshade);

// Display the Dynamic World visualization with the source Sentinel-2 image.
Map.setCenter(20.6729, 52.4305, 12);
Map.addLayer(
    s2Image, {min: 0, max: 3000, bands: ['B4', 'B3', 'B2']}, 'Sentinel-2 L1C');
Map.addLayer(
    dwRgbHillshade, {min: 0, max: 0.65}, 'Dynamic World V1 - label hillshade');

// // Unduh citra satelit (export to Google Drive)
Export.image.toDrive({
  image: dwRgbHillshade,
  description: 'halim_maret_2022_landcover',// nama file
  folder: 'pengdat-praktek',
  region: halim,
  scale: 10 //resolusi spasial, 1 piksel akan menggambarkan 10 meter
})
```

### 6. Mendapat nilai statistik dari gambar yang kita punya
- ke processing toolbox, double click zonal statistics
- pilih raster layer yg gambar TIF kita
- 1 zonal hanya sekali nilai
- misal raster bandnnya mau ambil yg merah (band 4)
- pilih lokasi polygon yang mau dikasih 1 nilai statistik
- pilih nilai statisitk yg mau diambil
- run, nanti liat hasilnya di shape, klik kanan `open attribute table`. nanti keluar ini
<img width="316" alt="image" src="https://github.com/latifaesmananda/try-satelite/assets/121326117/85e74f58-6bac-4007-9786-2222b4126635">
- terus bisa edit shape layer ganti warna supaya bisa keliatan kondisi vegetasinya
