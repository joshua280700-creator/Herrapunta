from flask import Flask, request, jsonify, send_file
import pandas as pd
import requests
import os
import io

app = Flask(__name__)

# --- CONFIGURACIÓN DE GOOGLE MAPS ---
GOOGLE_API_KEY = "AIzaSyC_r3iE-gyX_yBcApKof65behKsdR5c1lQ"

def geocodificar_con_google(direccion_sucia):
    query = f"{direccion_sucia}, Nuevo León, México"
    url = f"https://maps.googleapis.com/maps/api/geocode/json?address={query}&key={GOOGLE_API_KEY}"
    
    try:
        respuesta = requests.get(url).json()
        if respuesta['status'] == 'OK':
            resultado = respuesta['results'][0]
            direccion_completa = resultado['formatted_address']
            lat = resultado['geometry']['location']['lat']
            lng = resultado['geometry']['location']['lng']
            
            calle = ""
            numero = ""
            colonia = ""
            
            for componente in resultado['address_components']:
                tipos = componente['types']
                if 'route' in tipos: calle = componente['long_name']
                if 'street_number' in tipos: numero = componente['long_name']
                if 'sublocality' in tipos or 'neighborhood' in tipos: colonia = componente['long_name']
            
            calle_y_numero = f"{calle} {numero}".strip()
            if not colonia: colonia = "Área Local"
            
            return calle_y_numero, colonia, direccion_completa, lat, lng
            
    except Exception as e:
        print(f"Error consultando a Google: {e}")
        
    return direccion_sucia, "N/A", direccion_sucia, None, None

# --- INTERFAZ WEB CON RADAR DE FALLOS ---
HTML_PAGINA = """
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Torre de Control | Integración KML</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        body { font-family: 'Segoe UI', sans-serif; background: #f0f2f5; margin: 0; padding: 20px; display: flex; flex-direction: column; align-items: center; }
        .container { background: white; padding: 30px; border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,0.1); text-align: center; max-width: 700px; width: 100%; margin-bottom: 20px; }
        .file-box { border: 2px dashed #4285f4; padding: 25px; border-radius: 8px; background: #f1f3f4; cursor: pointer; margin-bottom: 20px; display: block; }
        button { background: #4285f4; color: white; border: none; padding: 12px; border-radius: 6px; cursor: pointer; width: 100%; font-weight: bold; font-size: 16px; margin-bottom: 10px; transition: 0.3s; }
        button:hover { background: #3367d6; }
        #btn-descargar { background: #0f9d58; display: none; }
        #btn-descargar:hover { background: #0b8043; }
        .status { margin-top: 15px; padding: 10px; border-radius: 4px; font-size: 14px; }
        .success { background: #dcfce3; color: #166534; font-weight: bold; }
        .warning-box { background: #fee2e2; color: #991b1b; padding: 15px; border-radius: 8px; text-align: left; margin-top: 15px; font-size: 13px; border: 1px solid #f87171;}
        #map { height: 450px; width: 100%; max-width: 900px; border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,0.1); display: none; border: 2px solid #4285f4; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Torre de Control Logística</h1>
        <p>Generador KML + Reporte de Extravíos</p>
        <label class="file-box" for="file-upload">
            <span id="file-name">Subir Viajes (.csv)</span>
            <input type="file" id="file-upload" style="display:none" accept=".csv" />
        </label>
        <button id="btn-procesar" disabled>Procesar Mapeo</button>
        <button id="btn-descargar">Descargar KML para My Maps</button>
        <div id="msg"></div>
    </div>
    
    <div id="map"></div>

    <script>
        const input = document.getElementById('file-upload');
        const btnP = document.getElementById('btn-procesar');
        const btnD = document.getElementById('btn-descargar');
        const msg = document.getElementById('msg');
        let kmlStringData = "";
        let map = null;

        input.onchange = (e) => { 
            if(e.target.files[0]) {
                document.getElementById('file-name').textContent = "📁 " + e.target.files[0].name;
                btnP.disabled = false;
            }
        };

        btnP.onclick = async () => {
            btnP.disabled = true;
            msg.textContent = "Analizando y Geocodificando...";
            msg.className = "status";
            const fd = new FormData();
            fd.append('archivo', input.files[0]);
            
            try {
                const res = await fetch('/procesar', { method: 'POST', body: fd });
                if(res.ok) {
                    const jsonResponse = await res.json();
                    kmlStringData = jsonResponse.kml_data;
                    
                    document.getElementById('map').style.display = 'block';
                    if(!map) {
                        map = L.map('map').setView([25.6750, -100.4616], 12);
                        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                            attribution: '© OpenStreetMap contributors'
                        }).addTo(map);
                    } else {
                        map.eachLayer((layer) => {
                            if (layer instanceof L.Marker) { map.removeLayer(layer); }
                        });
                    }

                    let pinesAgregados = 0;
                    const latlngs = [];

                    for(let paquete of jsonResponse.datos_mapa) {
                        L.marker([paquete.lat, paquete.lng]).addTo(map)
                         .bindPopup(`<b>${paquete.calle}</b><br>${paquete.colonia}`);
                        latlngs.push([paquete.lat, paquete.lng]);
                        pinesAgregados++;
                    }

                    if (latlngs.length > 0) {
                        map.fitBounds(L.latLngBounds(latlngs));
                    }

                    // --- SISTEMA DE ALERTAS EN PANTALLA ---
                    let htmlMensaje = "";
                    if(jsonResponse.fallidos.length > 0) {
                        htmlMensaje += `<div class="success" style="padding:10px;">✅ ${pinesAgregados} ubicados correctamente y listos en el KML.</div>`;
                        htmlMensaje += `<div class="warning-box">`;
                        htmlMensaje += `<b>⚠️ ATENCIÓN: Google no encontró coordenadas para ${jsonResponse.fallidos.length} paquetes:</b><ul style="margin-top:5px; padding-left:20px;">`;
                        
                        jsonResponse.fallidos.forEach(fallo => {
                            htmlMensaje += `<li><b>Fila Excel ${fallo.fila}:</b> ${fallo.direccion}</li>`;
                        });
                        
                        htmlMensaje += `</ul><i>*Estos paquetes fueron excluidos del KML. Revisa su ortografía e inténtalo manualmente.</i></div>`;
                    } else {
                        htmlMensaje = `<div class="success" style="padding:10px;">¡100% de Éxito! ${pinesAgregados} puntos listos para exportar. Ningún paquete extraviado.</div>`;
                    }

                    msg.innerHTML = htmlMensaje;
                    btnD.style.display = "block";
                } else {
                    msg.textContent = "Error al procesar el archivo.";
                }
            } catch (e) {
                msg.textContent = "Error de conexión con el servidor.";
                console.error(e);
            }
            btnP.disabled = false;
        };

        btnD.onclick = () => {
            const blob = new Blob([kmlStringData], { type: 'application/vnd.google-earth.kml+xml' });
            const a = document.createElement('a');
            a.href = URL.createObjectURL(blob);
            a.download = "Ruta_Mapeada_Detalle.kml";
            a.click();
        };
    </script>
</body>
</html>
"""

@app.route('/')
def inicio():
    return HTML_PAGINA

@app.route('/procesar', methods=['POST'])
def procesar_archivo():
    file = request.files['archivo']
    try:
        df = pd.read_csv(file, encoding='latin-1')
        col_dir = df.columns[0]
        
        datos_mapa = []
        fallidos = []
        
        kml_lines = [
            '<?xml version="1.0" encoding="UTF-8"?>',
            '<kml xmlns="http://www.opengis.net/kml/2.2">',
            '  <Document>',
            '    <name>Distribución de Viajes Oficial</name>'
        ]
        
        for index, row in df.iterrows():
            direccion_sucia = str(row[col_dir])
            calle, colonia, dir_completa, lat, lng = geocodificar_con_google(direccion_sucia)
            
            if lat and lng:
                datos_mapa.append({
                    'lat': lat,
                    'lng': lng,
                    'calle': calle,
                    'colonia': colonia
                })
                
                kml_lines.append('    <Placemark>')
                kml_lines.append(f'      <name><![CDATA[{calle}]]></name>')
                kml_lines.append('      <description><![CDATA[')
                kml_lines.append(f'        <h3 style="margin:0; color:#1a73e8;">Detalle del Paquete</h3><hr>')
                
                # LA MAGIA: Inyectamos TODAS las columnas del Excel original
                for col_name in df.columns:
                    valor = str(row[col_name])
                    if valor.lower() != "nan": # Ignoramos celdas vacías
                        kml_lines.append(f'        <b>{col_name}:</b> {valor}<br/>')
                        
                kml_lines.append(f'        <hr><b>Google Maps lo ubicó en:</b> {dir_completa}')
                kml_lines.append('      ]]></description>')
                kml_lines.append('      <Point>')
                kml_lines.append(f'        <coordinates>{lng},{lat},0</coordinates>')
                kml_lines.append('      </Point>')
                kml_lines.append('    </Placemark>')
            else:
                # Si Google falla, lo mandamos al "Radar de Rechazos"
                fallidos.append({
                    'fila': index + 2, # +2 para que coincida con el número de fila en Excel
                    'direccion': direccion_sucia
                })

        kml_lines.append('  </Document>')
        kml_lines.append('</kml>')
        
        return jsonify({
            "kml_data": "\n".join(kml_lines),
            "datos_mapa": datos_mapa,
            "fallidos": fallidos
        })
        
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
