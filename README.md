<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<title>Türkiye Camii Rehberi - Dijital Hayır Arşivi</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

<style>
:root {
    --ana-yesil: #14532d;
    --neon-altin: #fbbf24;
    --derin-arka-plan: #061f10;
}

html, body { height:100%; margin:0; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: var(--derin-arka-plan); color:#fff; overflow-x: hidden; }

/* Seccade Deseni Arka Planı */
body::before {
    content: "";
    position: fixed;
    top: 0; left: 0; width: 100%; height: 100%;
    background-image: url('https://www.transparenttextures.com/patterns/arabesque.png'); /* İnce motif */
    opacity: 0.1;
    z-index: -1;
}

header { 
  background: linear-gradient(135deg, #14532d 0%, #052c14 100%); 
  padding:20px; 
  border-bottom: 2px solid var(--neon-altin);
  text-align: center;
}

/* Neon Camii Simgesi */
.neon-camii {
    font-size: 40px;
    filter: drop-shadow(0 0 5px var(--neon-altin));
    margin-bottom: 10px;
    display: block;
}

header h1 { margin:0; font-size:22px; color: var(--neon-altin); letter-spacing: 1px; text-transform: uppercase; }

.mission-box {
  background: rgba(20, 83, 45, 0.8);
  padding: 15px;
  font-size: 13px;
  text-align: center;
  border-bottom: 1px solid rgba(251, 191, 36, 0.3);
}

.controls {
    padding: 15px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    background: rgba(0,0,0,0.2);
}

.search-row { display: flex; gap: 10px; }
input, select { 
    flex-grow: 1; 
    padding:12px; 
    border-radius:8px; 
    border:1px solid var(--ana-yesil); 
    background: #fff;
    color: #333;
}

.action-bar { display: flex; justify-content: space-between; padding: 5px 15px; }
.excel-btn { 
    background: #166534; 
    color: white; 
    border: 1px solid var(--neon-altin); 
    padding: 8px 15px; 
    border-radius: 20px; 
    font-size: 12px; 
    cursor: pointer;
}

#map { height: calc(100vh - 350px); width: 100%; border-top: 2px solid var(--neon-altin); }

/* Popup Tasarımı */
.popup-container { color: #333; min-width: 200px; }
.btn { display:block; text-align:center; padding:10px; margin-top:6px; border-radius:6px; text-decoration:none; font-weight:600; color:#fff; font-size:12px; }
.google { background:#ea4335; }
.yandex { background:#fc0; color:#000; }
.navigasyon { background:#3b82f6; } /* Mavi Yol Tarifi Butonu */
.osm { background:#14532d; border: 1px solid var(--neon-altin); margin-top: 10px; }
</style>
</head>
<body>

<header>
  <span class="neon-camii">🕌</span>
  <h1>Türkiye Camii Rehberi</h1>
  <small style="color:var(--neon-altin); opacity: 0.8;">Açık Veri Gönüllü Hareketi</small>
</header>

<div class="mission-box">
  <strong>Haritayı Birlikte Güçlendirelim</strong>
  İsimsiz camileri doğrulayarak hayra vesile olun.
</div>

<div class="controls">
  <div class="search-row">
    <select id="il">
      <option value="">İl Seçiniz...</option>
    </select>
    <input type="text" id="filterInput" placeholder="Cami veya İlçe Ara..." onkeyup="filterMarkers()">
  </div>
  <div class="action-bar">
    <span id="sayac">Bir il seçin...</span>
    <button class="excel-btn" onclick="exportToExcel()">📊 Excel Olarak İndir</button>
  </div>
</div>

<div id="map"></div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<script>
// Mevcut verilerin ve harita yapın korunuyor
const illerPlaka = ["Adana", "Adıyaman", "Afyonkarahisar", "Ağrı", "Amasya", "Ankara", "Antalya", "Artvin", "Aydın", "Balıkesir", "Bilecik", "Bingöl", "Bitlis", "Bolu", "Burdur", "Bursa", "Çanakkale", "Çankırı", "Çorum", "Denizli", "Diyarbakır", "Edirne", "Elazığ", "Erzincan", "Erzurum", "Eskişehir", "Gaziantep", "Giresun", "Gümüşhane", "Hakkari", "Hatay", "Isparta", "Mersin", "İstanbul", "İzmir", "Kars", "Kastamonu", "Kayseri", "Kırklareli", "Kırşehir", "Kocaeli", "Konya", "Kütahya", "Malatya", "Manisa", "Kahramanmaraş", "Mardin", "Muğla", "Muş", "Nevşehir", "Niğde", "Ordu", "Rize", "Sakarya", "Samsun", "Siirt", "Sinop", "Sivas", "Tekirdağ", "Tokat", "Trabzon", "Tunceli", "Şanlıurfa", "Uşak", "Van", "Yozgat", "Zonguldak", "Aksaray", "Bayburt", "Karaman", "Kırıkkale", "Batman", "Şırnak", "Bartın", "Ardahan", "Iğdır", "Yalova", "Karabük", "Kilis", "Osmaniye", "Düzce"];

const ilSelect = document.getElementById("il");
illerPlaka.sort().forEach((il) => { // Alfabetik sıralama ekledim
  const o = document.createElement("option");
  o.value = il;
  o.textContent = il;
  ilSelect.appendChild(o);
});

const map = L.map('map').setView([39,35], 6);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
const layer = L.layerGroup().addTo(map);
let allMarkers = []; // Arama için verileri tutacağız
let currentData = []; // Excel için

function fetchData() {
  const il = ilSelect.value;
  if (!il) return;
  layer.clearLayers();
  allMarkers = [];
  document.getElementById("sayac").innerText = "Veriler çekiliyor...";

  const query = `[out:json][timeout:50];area["name"="${il}"]["admin_level"="4"]->.a;(node["amenity"="place_of_worship"]["religion"="muslim"]["name"!~"."](area.a);way["amenity"="place_of_worship"]["religion"="muslim"]["name"!~"."](area.a););out center;`;

  fetch("https://overpass-api.de/api/interpreter", { method:"POST", body:query })
  .then(r=>r.json())
  .then(data=>{
    currentData = data.elements;
    document.getElementById("sayac").innerHTML = `<b>${data.elements.length}</b> İsimsiz Cami`;
    
    data.elements.forEach(el => {
      const lat = el.lat || el.center.lat;
      const lon = el.lon || el.center.lon;

      const popupHtml = `
      <div class="popup-container">
        <strong>📍 İsimsiz Cami</strong><br>
        <small>${lat}, ${lon}</small>
        <a class="btn google" target="_blank" href="https://www.google.com/maps?q=${lat},${lon}">🔍 Google Haritalar</a>
        <a class="btn navigasyon" target="_blank" href="https://www.google.com/maps/dir/?api=1&destination=${lat},${lon}">🚙 Yol Tarifi Al</a>
        <a class="btn yandex" target="_blank" href="https://yandex.com.tr/harita/?ll=${lon}%2C${lat}&z=19&l=stv%2Csta">🚗 Sokak Görünümü</a>
        <a class="btn osm" target="_blank" href="https://www.openstreetmap.org/edit?editor=id#map=19/${lat}/${lon}">✍️ İsim Ekle</a>
      </div>`;

      const marker = L.circleMarker([lat,lon], {
        radius:9, color:"#fbbf24", weight:2, fillColor:"#22c55e", fillOpacity:0.9
      }).bindPopup(popupHtml);
      
      marker.addTo(layer);
      allMarkers.push({marker, info: el});
    });
    if(allMarkers.length > 0) map.fitBounds(L.featureGroup(allMarkers.map(m => m.marker)).getBounds());
  });
}

// Yeni Özellik: Arama/Filtreleme
function filterMarkers() {
    const val = document.getElementById("filterInput").value.toLowerCase();
    allMarkers.forEach(item => {
        // Overpass verisinde isim olmadığı için koordinat veya ID bazlı arama yapılabilir
        // Eğer isim olsaydı item.info.tags.name üzerinden arama yapacaktık
        if (item.info.id.toString().includes(val)) {
            item.marker.addTo(layer);
        } else {
            layer.removeLayer(item.marker);
        }
    });
}

// Yeni Özellik: Excel Dışa Aktarma
function exportToExcel() {
    if (currentData.length === 0) return alert("Önce bir il seçin!");
    const worksheet = XLSX.utils.json_to_sheet(currentData.map(el => ({
        ID: el.id,
        Enlem: el.lat || el.center.lat,
        Boylam: el.lon || el.center.lon,
        Tip: el.type
    })));
    const workbook = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(workbook, worksheet, "Camiler");
    XLSX.writeFile(workbook, `${ilSelect.value}_isimsiz_camiler.xlsx`);
}

ilSelect.addEventListener("change", fetchData);
</script>
</body>
</html>
