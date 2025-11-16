<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSV Geodata Converter (CGCS2000 to WGS84)</title>
    <style>
        /* Custom Styles for Aesthetics */
        :root {
            --primary: #2196F3; /* Blue */
            --secondary: #607D8B; /* Grey-blue */
            --success: #4CAF50; /* Green */
            --danger: #F44336; /* Red */
            --background: #e3f2fd; /* Lightest blue */
            --card-bg: #ffffff;
            --shadow: 0 6px 15px rgba(0, 0, 0, 0.1);
        }
        body {
            font-family: 'Inter', sans-serif;
            margin: 0;
            padding: 40px 20px;
            background-color: var(--background);
            display: flex;
            justify-content: center;
            align-items: flex-start;
            min-height: 100vh;
        }
        .container {
            max-width: 800px;
            width: 100%;
            background: var(--card-bg);
            padding: 30px;
            border-radius: 12px;
            box-shadow: var(--shadow);
        }
        h2 {
            text-align: center;
            color: var(--primary);
            margin-bottom: 30px;
            font-weight: 600;
        }
        .input-group, .result-group {
            margin-bottom: 25px;
        }
        label {
            display: block;
            margin-bottom: 6px;
            font-weight: 500;
            color: #34495e;
            font-size: 0.95rem;
        }
        input[type="file"], input[type="number"], button, textarea, select, input[type="text"] {
            width: 100%;
            padding: 12px;
            margin-bottom: 10px;
            border: 1px solid #cfd8dc;
            border-radius: 6px;
            box-sizing: border-box;
            transition: border-color 0.3s;
        }
        input:focus, select:focus, textarea:focus {
            border-color: var(--primary);
            outline: none;
            box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.2);
        }
        textarea {
            resize: vertical;
            min-height: 250px;
            font-family: 'Consolas', 'Courier New', monospace;
            white-space: pre;
            overflow-x: scroll;
            font-size: 0.85rem;
            background-color: #f7f9fc;
        }
        button {
            background-color: var(--primary);
            color: white;
            cursor: pointer;
            font-weight: bold;
            font-size: 1rem;
            border: none;
            transition: background-color 0.3s ease, transform 0.1s ease, box-shadow 0.3s ease;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        button:hover {
            background-color: #1565C0;
            box-shadow: 0 6px 10px rgba(0, 0, 0, 0.15);
        }
        button:active {
            transform: translateY(1px);
        }
        .result-group {
            padding: 20px;
            border: 2px solid #e0f7fa;
            border-radius: 8px;
            background-color: #f1f8e9;
        }
        .result-group label {
            color: var(--success);
            font-size: 1rem;
        }
        .error-message {
            color: var(--danger);
            font-weight: 600;
            margin-top: 15px;
            padding: 10px;
            background-color: #ffebee;
            border-radius: 4px;
            border: 1px solid var(--danger);
            display: none; /* Hidden by default */
        }
        /* Responsive Columns for inputs */
        .column-select-group {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 20px;
        }
        @media (max-width: 600px) {
            .column-select-group {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>

<div class="container">
    <h2>📂 Geodata Converter: CGCS2000 (GK) to WGS84</h2>
    
    <div class="input-group">
        <label for="csvFile">1. Upload Coordinates CSV File:</label>
        <input type="file" id="csvFile" accept=".csv">
    </div>

    <div class="column-select-group">
        <div>
            <label for="xColumn">2. X/Easting Column Index (1-based):</label>
            <input type="number" id="xColumn" value="1" min="1">
        </div>
        <div>
            <label for="yColumn">3. Y/Northing Column Index (1-based):</label>
            <input type="number" id="yColumn" value="2" min="1">
        </div>
    </div>
    
    <div class="column-select-group">
        <div>
            <label for="zone">4. Gaussian-Kruger Zone (25-45):</label>
            <input type="number" id="zone" value="40" min="25" max="45">
        </div>
        <div>
            <label for="separator">5. CSV Separator (e.g., , or ;):</label>
            <input type="text" id="separator" value="," maxlength="1">
        </div>
    </div>
    
    <div class="input-group">
        <label for="epsg">6. EPSG Code (4499 or 4500):</label>
        <select id="epsg">
            <option value="4499">4499 (CGCS2000)</option>
            <option value="4500" selected>4500 (CGCS2000)</option>
        </select>
    </div>

    <button onclick="loadFileAndProcess()">Convert CSV Data and Locate Cities</button>
    <p style="text-align: center; color: var(--secondary); font-size: 0.9rem;">
        **Tip:** Press F12 or Ctrl+Shift+I to open the browser Console for debugging details.
    </p>

    <div id="errorMessage" class="error-message"></div>

    <div class="result-group">
        <label>Conversion Results (Original data + 4 new columns):</label>
        <textarea id="outputData" readonly placeholder="Upload a file and click 'Convert' to see the results. The output will be a CSV-compatible text that includes WGS84 coordinates and the inferred Province/City."></textarea>
    </div>
</div>

<script>
    // Constants for CGCS2000 (China Geodetic Coordinate System 2000) Ellipsoid Parameters
    // Based on CGCS2000 standard
    const A = 6378137.0; // Semi-major axis
    const F = 1 / 298.257222101; // Flattening
    const PI = Math.PI;
    const FALSE_EASTING = 500000.0; // Standard false easting for GK projection

    /**
     * Converts Gaussian-Kruger (GK, CGCS2000) coordinates to WGS84 Latitude and Longitude.
     * This function implements the reverse projection formula.
     * @param {number} X - Easting coordinate (m)
     * @param {number} Y - Northing coordinate (m)
     * @param {number} Zone - The 3-degree GK zone number (25 to 45)
     * @returns {{lat: number, lon: number} | null} - Object containing WGS84 Lat/Lon, or null on error.
     */
    function convertGK4500(X, Y, Zone) {
        if (Zone < 25 || Zone > 45 || isNaN(X) || isNaN(Y)) return null;

        // Calculate Longitude of Central Meridian (L0) in degrees and radians
        const lon0Deg = Zone * 3;
        const lon0Rad = lon0Deg * PI / 180;
        
        // Remove zone and false easting from Easting (X) to get true distance from L0
        // X is the Easting coordinate in meters, Y is the Northing coordinate in meters.
        // The standard GK Easting format is: (Zone Number) * 1,000,000 + 500,000 + (True Easting)
        // Since we only have the standard Zone number (e.g., 40), we assume the full Easting
        // includes the 1,000,000 * Zone part.
        const X0 = X - Zone * 1000000.0; 
        
        // Calculate the True Easting (distance from Central Meridian) 
        const trueEasting = X0 - FALSE_EASTING;
        if (trueEasting > 250000 || trueEasting < -250000) {
            console.warn(`[Warning] True Easting (${trueEasting.toFixed(2)}m) is outside typical 3-degree GK limits.`);
        }

        // Derived Ellipsoid Constants
        const e2 = 2 * F - F * F; // First eccentricity squared
        const ee = e2 / (1 - e2); // Second eccentricity squared

        // Calculate Footpoint Latitude (fp) iteration start (mu)
        // mu is the distance from the equator divided by the radius of curvature at the equator
        const mu = Y / (A * (1 - e2 / 4 - 3 * Math.pow(e2, 2) / 64 - 5 * Math.pow(e2, 3) / 256));
        
        // Iteration constant
        const e1 = (1 - Math.sqrt(1 - e2)) / (1 + Math.sqrt(1 - e2));

        // Refine Footpoint Latitude (fp) using closed-form series (often preferred over iterative)
        let fp = mu + (3 * e1 / 2 - 27 * Math.pow(e1, 3) / 32) * Math.sin(2 * mu);
        fp += (21 * Math.pow(e1, 2) / 16 - 55 * Math.pow(e1, 4) / 32) * Math.sin(4 * mu);
        fp += (151 * Math.pow(e1, 3) / 96) * Math.sin(6 * mu);
        fp += (1097 * Math.pow(e1, 4) / 512) * Math.sin(8 * mu);

        const sinfp = Math.sin(fp);
        const cosfp = Math.cos(fp);
        const tanfp = Math.tan(fp);

        // N1 (Radius of Curvature in Prime Vertical), R1 (Meridian Radius of Curvature)
        const N1 = A / Math.sqrt(1 - e2 * Math.pow(sinfp, 2));
        const R1 = A * (1 - e2) / Math.pow((1 - e2 * Math.pow(sinfp, 2)), 1.5);

        // C1 and T1 terms for series
        const C1 = ee * Math.pow(cosfp, 2);
        const T1 = Math.pow(tanfp, 2);

        // D is normalized distance from Central Meridian
        const D = trueEasting / N1;

        // Calculate Latitude (phi) in radians using 6th-order series
        let latRad = fp - (N1 * tanfp / R1) * (
            Math.pow(D, 2) / 2 - 
            (5 + 3 * T1 + 10 * C1 - 4 * Math.pow(C1, 2) - 9 * ee) * Math.pow(D, 4) / 24 + 
            (61 + 90 * T1 + 298 * C1 + 45 * Math.pow(T1, 2) - 252 * ee - 3 * Math.pow(C1, 2)) * Math.pow(D, 6) / 720
        );

        // Calculate Longitude difference (delta_lambda) in radians using 5th-order series
        let lonRad = (
            D - (1 + 2 * T1 + C1) * Math.pow(D, 3) / 6 +
            (5 - 2 * C1 + 28 * T1 - 3 * Math.pow(C1, 2) + 8 * ee + 24 * Math.pow(T1, 2)) * Math.pow(D, 5) / 120
        ) / cosfp;

        // Final Longitude calculation
        const lonFinalRad = lon0Rad + lonRad;
        
        // Convert to Degrees and round to 6 decimal places for precision
        const latDeg = latRad * 180 / PI;
        const lonDeg = lonFinalRad * 180 / PI;

        const lat = Math.round(latDeg * 1000000) / 1000000;
        const lon = Math.round(lonDeg * 1000000) / 1000000;

        return { lat, lon };
    }


    /**
     * Performs a rough geographic lookup based on WGS84 coordinates 
     * using pre-defined bounding boxes from the original VBA code logic.
     * @param {number} lat - WGS84 Latitude
     * @param {number} lon - WGS84 Longitude
     * @returns {{province: string, city: string}} - Inferred location.
     */
    function convertXYToCity(lat, lon) {
        let province = "Unknown";
        let city = "Unknown";

        // --- BOUNDING BOX LOGIC (Directly ported) ---
        // Jilin Province
        if (lat >= 43.5 && lat <= 43.56 && lon >= 127.05 && lon <= 127.1) {
            province = "Jilin"; city = "Jiaohe (Songjiangzhen, Jilin City)";
        } else if (lat >= 44.5 && lat <= 44.57 && lon >= 126.75 && lon <= 126.85) {
            province = "Jilin"; city = "Yushu (Changchun)";
        } else if (lat >= 45.03 && lat <= 45.09 && lon >= 126.53 && lon <= 126.6) {
            province = "Jilin"; city = "Yushu";
        } else if (lat >= 42.1 && lat <= 42.3 && lon >= 127.3 && lon <= 127.6) {
            province = "Jilin"; city = "Baishan";
        } else if (lat >= 42.2 && lat <= 42.3 && lon >= 127.6 && lon <= 127.7) {
            province = "Jilin"; city = "Baishan";
        } else if (lat >= 42.2 && lat <= 42.4 && lon >= 127.6 && lon <= 127.9) {
            province = "Jilin"; city = "Baishan";
        } else if (lat >= 43.52 && lat <= 43.56 && lon >= 124.32 && lon <= 124.36) {
            province = "Jilin"; city = "Lishu County";
        } else if (lat >= 43.3 && lat <= 43.4 && lon >= 124.8 && lon <= 125.0) {
            province = "Jilin"; city = "Dongliao County";
        } else if (lat >= 43.3 && lat <= 43.4 && lon >= 124.0 && lon <= 124.1) {
            province = "Jilin"; city = "Meihekou";
        } else if (lat >= 42.5 && lat <= 42.7 && lon >= 128.0 && lon <= 128.4) {
            province = "Jilin"; city = "Antu County";
        } else if (lat >= 43.2 && lat <= 43.3 && lon >= 127.9 && lon <= 128.1) {
            province = "Jilin"; city = "Dunhua";
        } else if (lat >= 44.0 && lat <= 44.2 && lon >= 125.2 && lon <= 125.6) {
            province = "Jilin"; city = "Yitong Manchu County";
        } else if (lat >= 45.3 && lat <= 45.4 && lon >= 122.7 && lon <= 122.9) {
            province = "Jilin"; city = "Taonan";
        } else if (lat >= 43.2 && lat <= 43.3 && lon >= 128.7 && lon <= 128.9) {
            province = "Jilin"; city = "Yanbian";
        } else if (lat >= 43.22 && lat <= 43.26 && lon >= 129.2 && lon <= 129.23) {
            province = "Jilin"; city = "Yanbian";
        } else if (lat >= 43.23 && lat <= 43.25 && lon >= 129.21 && lon <= 129.24) {
            province = "Jilin"; city = "Hunchun";
        } else if (lat >= 43.23 && lat <= 43.27 && lon >= 129.19 && lon <= 129.22) {
            province = "Jilin"; city = "Tumen";
        } else if (lat >= 42.7 && lat <= 42.8 && lon >= 123.5 && lon <= 123.7) {
            province = "Jilin"; city = "Siping";
        } else if (lat >= 44.9 && lat <= 45.0 && lon >= 125.4 && lon <= 125.6) {
            province = "Jilin"; city = "Yushu";
        } else if (lat >= 43.5 && lat <= 43.6 && lon >= 127.3 && lon <= 127.4) {
            province = "Jilin"; city = "Dunhua";
        } else if (lat >= 42.25 && lat <= 42.5 && lon >= 128.25 && lon <= 128.6) {
            city = "Baihe"; province = "Jilin";
        } else if (lat >= 43.34 && lat <= 43.35 && lon >= 127.03 && lon <= 127.05) {
            city = "Dongfeng County"; province = "Jilin";
        } else if (lat >= 43.5 && lat <= 43.6 && lon >= 124.3 && lon <= 124.4) {
            province = "Jilin"; city = "Dongliao County";
        } else if (lat >= 43.0 && lat <= 43.1 && lon >= 129.6 && lon <= 129.7) {
            province = "Jilin"; city = "Tumen";
        } else if (lat >= 43.0 && lat <= 43.2 && lon >= 129.5 && lon <= 130.0) {
            province = "Jilin"; city = "Yanbian";
        } else if (lat >= 42.9 && lat <= 43.1 && lon >= 130.9 && lon <= 131.0) {
            city = "Hunchun"; province = "Jilin";
        } else if (lat >= 43.27 && lat <= 43.35 && lon >= 126.7 && lon <= 126.9) {
            city = "Huadian"; province = "Jilin";
        } else if (lat >= 42.735 && lat <= 42.745 && lon >= 127.45 && lon <= 127.47) {
            province = "Jilin"; city = "Huadian";
        } else if (lat >= 42.23 && lat <= 42.25 && lon >= 128.28 && lon <= 128.31) {
            province = "Jilin"; city = "Baihe";
        } else if (lat >= 44.25 && lat <= 44.3 && lon >= 127.05 && lon <= 127.14) {
            province = "Jilin"; city = "Shulan";
        }
        
        // Heilongjiang Province
        else if (lat >= 45.08 && lat <= 45.2 && lon >= 127.65 && lon <= 128.2) {
            province = "Heilongjiang"; city = "Shangzhi (Harbin)";
        } else if (lat >= 44.86 && lat <= 44.88 && lon >= 128.34 && lon <= 128.36) {
            province = "Heilongjiang"; city = "Shangzhi (Zhenzhushanxiang)";
        } else if (lat >= 44.6 && lat <= 44.7 && lon >= 131.0 && lon <= 131.1) {
            province = "Heilongjiang"; city = "Mishan (Jixi)";
        } else if (lat >= 45.629 && lat <= 45.75 && lon >= 131.97 && lon <= 132.18) {
            province = "Heilongjiang"; city = "Mishan (Jixi)";
        } else if (lat >= 46.48 && lat <= 46.75 && lon >= 132.83 && lon <= 133.12) {
            province = "Heilongjiang"; city = "Shuangyashan";
        } else if (lat >= 46.78 && lat <= 46.87 && lon >= 131.71 && lon <= 132.1) {
            province = "Heilongjiang"; city = "Youyi County (Shuangyashan)";
        } else if (lat >= 46.67 && lat <= 46.69 && lon >= 132.95 && lon <= 132.97) {
            province = "Heilongjiang"; city = "Shuangyashan, Baoqing County";
        } else if (lat >= 46.6 && lat <= 46.7 && lon >= 131.9 && lon <= 132.1) {
            province = "Heilongjiang"; city = "Jidong County";
        } else if (lat >= 47.9 && lat <= 48.1 && lon >= 124.6 && lon <= 124.8) {
            province = "Heilongjiang"; city = "Nenjiang (Heihe)";
        } else if (lat >= 47.0 && lat <= 48.0 && lon >= 124.0 && lon <= 124.5) {
            province = "Heilongjiang"; city = "Nehe (Qiqihar)";
        } else if (lat >= 46.5 && lat <= 46.7 && lon >= 131.8 && lon <= 132.0) {
            province = "Heilongjiang"; city = "Jixi";
        } else if (lat >= 47.1 && lat <= 47.2 && lon >= 131.3 && lon <= 131.6) {
            province = "Heilongjiang"; city = "Hegang";
        } else if (lat >= 44.2 && lat <= 44.4 && lon >= 127.0 && lon <= 127.2) {
            province = "Heilongjiang"; city = "Hailin";
        } else if (lat >= 47.12 && lat <= 47.35 && lon >= 132.55 && lon <= 132.75) {
            province = "Heilongjiang"; city = "Fujin (Jiamusi)";
        } else if (lat >= 44.4 && lat <= 44.6 && lon >= 128.3 && lon <= 128.6) {
            province = "Heilongjiang"; city = "Mudanjiang";
        } else if (lat >= 47.3 && lat <= 47.5 && lon >= 127.6 && lon <= 127.8) {
            province = "Heilongjiang"; city = "Yichun";
        } else if (lat >= 43.53 && lat <= 43.55 && lon >= 130.33 && lon <= 130.35) {
            province = "Heilongjiang"; city = "Mishan";
        } else if (lat >= 45.8 && lat <= 46.1 && lon >= 126.9 && lon <= 127.1) {
            province = "Heilongjiang"; city = "Harbin";
        } else if (lat >= 47.36 && lat <= 47.4 && lon >= 133.66 && lon <= 133.72) {
            province = "Heilongjiang"; city = "Luobei";
        } else if (lat >= 47.2 && lat <= 47.35 && lon >= 126.5 && lon <= 126.7) {
            province = "Heilongjiang"; city = "Suihua";
        }

        // Jilin (additional cases)
        else if (lat >= 43.1 && lat <= 43.13 && lon >= 127.4 && lon <= 127.43) {
            province = "Jilin"; city = "Panshi";
        } else if (lat >= 43.63 && lat <= 43.66 && lon >= 126.58 && lon <= 126.6) {
            province = "Jilin"; city = "Huadian";
        }
        
        // Liaoning
        else if (lat >= 41.7 && lat <= 41.9 && lon >= 123.3 && lon <= 123.6) {
            province = "Liaoning"; city = "Shenyang";
        } else if (lat >= 38.8 && lat <= 39.2 && lon >= 121.4 && lon <= 121.8) {
            province = "Liaoning"; city = "Dalian";
        }

        // Inner Mongolia
        else if (lat >= 46.3 && lat <= 46.6 && lon >= 121.9 && lon <= 122.2) {
            province = "Inner Mongolia"; city = "Horqin Right Front Banner";
        } else if (lat >= 46.1 && lat <= 46.3 && lon >= 122.0 && lon <= 122.3) {
            province = "Inner Mongolia"; city = "Ulanhot";
        } else if (lat >= 46.4 && lat <= 46.6 && lon >= 120.8 && lon <= 121.1) {
            province = "Inner Mongolia"; city = "Ulanhot";
        } else if (lat >= 43.67 && lat <= 43.69 && lon >= 118.84 && lon <= 118.86) {
            province = "Inner Mongolia"; city = "Chifeng, Bairin Right Banner";
        }

        // Shandong Province
        else if (lat >= 36.35 && lat <= 36.4 && lon >= 116.43 && lon <= 116.47) {
            province = "Shandong"; city = "Dong'e County (Liaocheng)";
        } else if (lat >= 36.6 && lat <= 36.8 && lon >= 117.0 && lon <= 117.2) {
            province = "Shandong"; city = "Jinan";
        } else if (lat >= 35.0 && lat <= 35.3 && lon >= 118.2 && lon <= 118.5) {
            province = "Shandong"; city = "Linyi";
        } else if (lat >= 36.0 && lat <= 36.2 && lon >= 120.3 && lon <= 120.5) {
            province = "Shandong"; city = "Qingdao";
        } else if (lat >= 37.4 && lat <= 37.6 && lon >= 121.3 && lon <= 121.5) {
            province = "Shandong"; city = "Yantai";
        } else if (lat >= 37.3 && lat <= 37.5 && lon >= 122.0 && lon <= 122.3) {
            province = "Shandong"; city = "Weihai";
        } else if (lat >= 36.6 && lat <= 36.8 && lon >= 119.9 && lon <= 120.1) {
            province = "Shandong"; city = "Weifang";
        } else if (lat >= 35.4 && lat <= 35.6 && lon >= 116.9 && lon <= 117.1) {
            province = "Shandong"; city = "Zaozhuang";
        } else if (lat >= 36.0 && lat <= 36.2 && lon >= 117.1 && lon <= 117.3) {
            province = "Shandong"; city = "Tai'an";
        } else if (lat >= 37.3 && lat <= 37.5 && lon >= 116.8 && lon <= 117.0) {
            province = "Shandong"; city = "Dezhou";
        } else if (lat >= 36.7 && lat <= 36.9 && lon >= 118.0 && lon <= 118.2) {
            province = "Shandong"; city = "Zibo";
        } else if (lat >= 37.4 && lat <= 37.6 && lon >= 118.6 && lon <= 118.8) {
            province = "Shandong"; city = "Dongying";
        } else if (lat >= 35.5 && lat <= 35.7 && lon >= 119.1 && lon <= 119.3) {
            province = "Shandong"; city = "Rizhao";
        } else if (lat >= 36.7 && lat <= 36.9 && lon >= 121.1 && lon <= 121.3) {
            province = "Shandong"; city = "Laizhou (Yantai)";
        } else if (lat >= 36.1 && lat <= 36.3 && lon >= 119.9 && lon <= 120.1) {
            province = "Shandong"; city = "Jiaozhou (Qingdao)";
        }
        // Shandong fallback (must be LAST before Case Else)
        else if (lat >= 34.2 && lat <= 38.9 && lon >= 116.0 && lon <= 122.7) {
            province = "Shandong"; city = "Unknown";
        }

        return { province, city };
    }


    // --- CSV HANDLING LOGIC ---

    /**
     * Shows error message in the dedicated div.
     * @param {string} message 
     */
    function displayError(message) {
        const errorElement = document.getElementById('errorMessage');
        errorElement.textContent = message;
        errorElement.style.display = message ? 'block' : 'none';
        document.getElementById('outputData').value = '';
    }

    /**
     * Initializes the file reading process.
     */
    function loadFileAndProcess() {
        displayError(''); // Clear previous errors
        const fileInput = document.getElementById('csvFile');
        const file = fileInput.files[0];

        if (!file) {
            displayError("🔴 Please select a CSV file to upload.");
            return;
        }

        const reader = new FileReader();
        reader.onload = function(event) {
            const csvContent = event.target.result;
            processCsv(csvContent);
        };
        reader.onerror = function() {
            displayError("🔴 Error reading file. Please check file permissions or format.");
        };
        reader.readAsText(file);
    }

    /**
     * Parses the CSV content, performs conversion, and generates the output CSV string.
     * @param {string} csvText - The raw content of the uploaded CSV file.
     */
    function processCsv(csvText) {
        const zone = parseInt(document.getElementById('zone').value);
        const epsg = document.getElementById('epsg').value;
        const xColIndex = parseInt(document.getElementById('xColumn').value) - 1; // 0-based
        const yColIndex = parseInt(document.getElementById('yColumn').value) - 1; // 0-based
        const separator = document.getElementById('separator').value || ','; // Get user-defined separator
        const outputData = document.getElementById('outputData');

        console.log(`--- Starting Conversion ---`);
        console.log(`Zone: ${zone}, X Col: ${xColIndex + 1}, Y Col: ${yColIndex + 1}, Separator: "${separator}"`);

        if (epsg != 4499 && epsg != 4500) {
            displayError("🔴 Unsupported EPSG code. Only 4499 and 4500 are supported.");
            return;
        }

        if (isNaN(zone) || zone < 25 || zone > 45) {
            displayError("🔴 Invalid Zone value. Must be an integer between 25 and 45.");
            return;
        }
        
        // Split into lines and filter out empty lines
        const lines = csvText.trim().split('\n').filter(line => line.trim() !== '');
        
        if (lines.length === 0) {
            displayError("🔴 CSV file is empty.");
            return;
        }

        const headerLine = lines[0].trim();
        const bodyLines = lines.slice(1);

        // 1. Process Header
        let headerParts = headerLine.split(separator).map(p => p.trim());
        
        // Validation check for column indices
        if (xColIndex >= headerParts.length || yColIndex >= headerParts.length || xColIndex < 0 || yColIndex < 0) {
             displayError(`🔴 Invalid Column Index. Max columns detected: ${headerParts.length}. Please check X (${xColIndex + 1}) and Y (${yColIndex + 1}) settings.`);
             return;
        }
        console.log(`Detected columns: ${headerParts.length}. Header: ${headerParts.join(' | ')}`);


        // Add new header columns
        headerParts.push('WGS84_LAT_DEG', 'WGS84_LON_DEG', 'PROVINCE', 'CITY');
        const outputLines = [headerParts.join(separator)];

        // 2. Process Body Lines
        for (let i = 0; i < bodyLines.length; i++) {
            const line = bodyLines[i].trim();
            // IMPORTANT: Use the specified separator for splitting the data line
            let parts = line.split(separator).map(p => p.trim());

            // Check if the row has enough parts
            if (parts.length < Math.max(xColIndex, yColIndex) + 1) {
                outputLines.push(parts.join(separator) + separator.repeat(4) + 'ERROR_INCOMPLETE_ROW');
                console.error(`Row ${i + 1} Error: Incomplete row data. Line length: ${parts.length}. Skipped.`);
                continue;
            }

            const X = parseFloat(parts[xColIndex]);
            const Y = parseFloat(parts[yColIndex]);

            console.log(`Processing row ${i + 1}. Raw X/Y values: "${parts[xColIndex]}", "${parts[yColIndex]}". Parsed X: ${X}, Y: ${Y}`);

            // Check for valid numbers in coordinate columns
            if (isNaN(X) || isNaN(Y)) {
                const errorMsg = 'Non-numeric coordinate in source column.';
                outputLines.push(parts.join(separator) + separator.repeat(4) + 'ERROR_NON_NUMERIC');
                console.error(`Row ${i + 1} Error: ${errorMsg}. X index: ${xColIndex+1}, Y index: ${yColIndex+1}`);
                continue;
            }

            const wgs84 = convertGK4500(X, Y, zone);
            
            if (!wgs84) {
                // Conversion failed (e.g., coordinate out of range for the zone)
                const errorMsg = 'Invalid Conversion (Check X/Y range for Zone ' + zone + ')';
                outputLines.push(parts.join(separator) + separator.repeat(4) + 'ERROR_CONVERSION_FAIL');
                console.error(`Row ${i + 1} Error: ${errorMsg}`);
                continue;
            }

            // Conversion successful: Perform Location Lookup
            const location = convertXYToCity(wgs84.lat, wgs84.lon);
            console.log(`Row ${i + 1} Success: WGS84 Lat/Lon: ${wgs84.lat.toFixed(6)}, ${wgs84.lon.toFixed(6)}. Loc: ${location.province}/${location.city}`);

            // Append results to the current row, rounded to 6 decimals
            const latStr = wgs84.lat.toFixed(6);
            const lonStr = wgs84.lon.toFixed(6);
            
            parts.push(latStr, lonStr, location.province, location.city);
            outputLines.push(parts.join(separator));
        }

        // 3. Display Result
        outputData.value = outputLines.join('\n');
    }
    
    // Attach to global scope for button click
    window.loadFileAndProcess = loadFileAndProcess;

</script>

</body>
</html>
