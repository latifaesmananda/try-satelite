# try-satelite

### 1. Sentinel-2 Halim, without preprocessing. Only median
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
