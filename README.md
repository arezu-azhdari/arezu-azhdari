# business_intelligence_system.py - کد کامل و یکپارچه

import re
import time
import json
import sqlite3
import requests
from bs4 import BeautifulSoup
from typing import List, Dict
from urllib.parse import urljoin, urlparse
from datetime import datetime
from flask import Flask, render_template, request, jsonify
import pandas as pd

# ==================== بخش اول: کرالر و استخراج اطلاعات ====================

class BusinessContactExtractor:
    def __init__(self):
        self.headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        }
        
    def search_businesses(self, topic: str, location: str = "") -> List[Dict]:
        """جستجوی کسب و کارها در منابع مختلف"""
        all_businesses = []
        
        # جستجو در دایرکتوری‌های عمومی
        directories = [
            f"https://www.yellowpages.com/search?search_terms={topic.replace(' ', '+')}",
            f"https://www.superpages.com/search?search_terms={topic.replace(' ', '+')}"
        ]
        
        for directory in directories:
            businesses = self.crawl_directory(directory)
            all_businesses.extend(businesses)
            time.sleep(1)
        
        # جستجوی مستقیم گوگل
        google_businesses = self.google_search(topic)
        all_businesses.extend(google_businesses)
        
        # حذف تکراری‌ها بر اساس شماره تلفن
        unique_businesses = {}
        for biz in all_businesses:
            phone = biz.get('phone', '')
            if phone and phone not in unique_businesses:
                unique_businesses[phone] = biz
            elif not phone and biz.get('name'):
                unique_businesses[biz.get('name', str(time.time()))] = biz
        
        return list(unique_businesses.values())
    
    def crawl_directory(self, url: str) -> List[Dict]:
        """استخراج اطلاعات از دایرکتوری‌ها"""
        businesses = []
        try:
            response = requests.get(url, headers=self.headers, timeout=15)
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # پیدا کردن کارت‌های کسب و کار
            cards = soup.find_all('div', class_=re.compile(r'result|listing|business'))
            
            for card in cards[:30]:
                business = self.extract_from_element(card, url)
                if business.get('phone') or business.get('email'):
                    businesses.append(business)
                    
        except Exception as e:
            print(f"Error: {e}")
        
        return businesses
    
    def extract_from_element(self, element, base_url: str) -> Dict:
        """استخراج اطلاعات از یک المان HTML"""
        text = element.get_text().lower()
        
        # استخراج نام (معمولا در تگ h2 یا strong)
        name = ""
        for name_tag in ['h2', 'h3', 'strong', '.business-name']:
            tag = element.find(name_tag)
            if tag:
                name = tag.get_text().strip()
                break
        
        # الگوهای شماره تلفن
        phone_patterns = [
            r'\+98[0-9]{10}',
            r'0[0-9]{2,3}[0-9]{8}',
            r'\+1[0-9]{10}',
            r'\+44[0-9]{10}',
            r'[0-9]{3}[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}'
        ]
        
        phone = ""
        for pattern in phone_patterns:
            phones = re.findall(pattern, text)
            if phones:
                phone = self.clean_phone(phones[0])
                break
        
        # استخراج ایمیل
        email_pattern = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
        emails = re.findall(email_pattern, text)
        email = emails[0] if emails else ""
        
        # استخراج وبسایت
        website = ""
        links = element.find_all('a', href=True)
        for link in links:
            href = link['href']
            if 'http' in href and ('.com' in href or '.org' in href or '.net' in href):
                if 'facebook' not in href and 'twitter' not in href:
                    website = href
                    break
        
        return {
            'name': name[:100] if name else "نامشخص",
            'phone': phone,
            'email': email,
            'website': website,
            'source': base_url
        }
    
    def google_search(self, query: str) -> List[Dict]:
        """جستجوی ساده در گوگل (بدون API)"""
        businesses = []
        
        search_url = f"https://www.google.com/search?q={query.replace(' ', '+')}+contact+phone+email"
        
        try:
            response = requests.get(search_url, headers=self.headers, timeout=10)
            soup = BeautifulSoup(response.text, 'html.parser')
            
            # پیدا کردن نتایج جستجو
            results = soup.find_all('div', class_=re.compile(r'g|rc'))
            
            for result in results[:20]:
                # استخراج لینک
                link_tag = result.find('a', href=True)
                if link_tag:
                    url = link_tag['href']
                    if url.startswith('/url?q='):
                        url = url[7:].split('&')[0]
                    
                    title_tag = result.find('h3')
                    name = title_tag.get_text() if title_tag else "نامشخص"
                    
                    # بررسی صفحه برای اطلاعات تماس
                    contact_info = self.extract_from_page(url)
                    
                    businesses.append({
                        'name': name[:100],
                        'phone': contact_info.get('phone', ''),
                        'email': contact_info.get('email', ''),
                        'website': url,
                        'source': 'Google Search'
                    })
                    
                    time.sleep(0.5)
                    
        except Exception as e:
            print(f"Google search error: {e}")
        
        return businesses
    
    def extract_from_page(self, url: str) -> Dict:
        """استخراج اطلاعات تماس از یک صفحه وب"""
        info = {'phone': '', 'email': ''}
        
        try:
            response = requests.get(url, headers=self.headers, timeout=10)
            soup = BeautifulSoup(response.text, 'html.parser')
            text = soup.get_text().lower()
            
            # جستجوی شماره تلفن
            phone_patterns = [
                r'\+98[0-9]{10}',
                r'0[0-9]{2,3}[0-9]{8}',
                r'[0-9]{3}[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}'
            ]
            
            for pattern in phone_patterns:
                phones = re.findall(pattern, text)
                if phones:
                    info['phone'] = self.clean_phone(phones[0])
                    break
            
            # جستجوی ایمیل
            email_pattern = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
            emails = re.findall(email_pattern, text)
            if emails:
                info['email'] = emails[0]
                
        except:
            pass
        
        return info
    
    def clean_phone(self, phone: str) -> str:
        """پاکسازی شماره تلفن"""
        phone = re.sub(r'[^0-9+]', '', phone)
        return phone

# ==================== بخش دوم: دیتابیس ====================

class DatabaseManager:
    def __init__(self, db_name="business_leads.db"):
        self.conn = sqlite3.connect(db_name)
        self.create_tables()
    
    def create_tables(self):
        cursor = self.conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS businesses (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                phone TEXT,
                email TEXT,
                website TEXT,
                search_topic TEXT,
                created_at TIMESTAMP,
                contacted BOOLEAN DEFAULT 0,
                notes TEXT
            )
        ''')
        self.conn.commit()
    
    def save_businesses(self, businesses: List[Dict], topic: str):
        cursor = self.conn.cursor()
        saved_count = 0
        
        for biz in businesses:
            if biz.get('phone') or biz.get('email'):
                cursor.execute('''
                    INSERT OR IGNORE INTO businesses 
                    (name, phone, email, website, search_topic, created_at)
                    VALUES (?, ?, ?, ?, ?, ?)
                ''', (
                    biz.get('name', ''),
                    biz.get('phone', ''),
                    biz.get('email', ''),
                    biz.get('website', ''),
                    topic,
                    datetime.now()
                ))
                saved_count += 1
        
        self.conn.commit()
        return saved_count
    
    def get_all_businesses(self, topic=None):
        cursor = self.conn.cursor()
        if topic:
            cursor.execute("SELECT * FROM businesses WHERE search_topic = ?", (topic,))
        else:
            cursor.execute("SELECT * FROM businesses")
        
        columns = [description[0] for description in cursor.description]
        businesses = []
        for row in cursor.fetchall():
            businesses.append(dict(zip(columns, row)))
        
        return businesses
    
    def update_contact_status(self, business_id, contacted=True, notes=""):
        cursor = self.conn.cursor()
        cursor.execute('''
            UPDATE businesses 
            SET contacted = ?, notes = ?
            WHERE id = ?
        ''', (contacted, notes, business_id))
        self.conn.commit()
    
    def export_to_excel(self, filename="business_contacts.xlsx"):
        df = pd.read_sql_query("SELECT * FROM businesses", self.conn)
        df.to_excel(filename, index=False)
        return filename
    
    def close(self):
        self.conn.close()

# ==================== بخش سوم: وب اپلیکیشن (Flask) ====================

app = Flask(__name__)
db = DatabaseManager()
extractor = BusinessContactExtractor()

# HTML تمپلیت (به صورت رشته)
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html dir="rtl" lang="fa">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>سیستم هوشمند جستجوی کسب و کار</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Tahoma', 'Arial', sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            overflow: hidden;
        }
        
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }
        
        .header h1 {
            font-size: 28px;
            margin-bottom: 10px;
        }
        
        .header p {
            opacity: 0.9;
        }
        
        .search-box {
            padding: 30px;
            background: #f8f9fa;
            border-bottom: 1px solid #e0e0e0;
        }
        
        .search-form {
            display: flex;
            gap: 15px;
            flex-wrap: wrap;
        }
        
        .search-form input {
            flex: 1;
            padding: 15px;
            border: 2px solid #ddd;
            border-radius: 10px;
            font-size: 16px;
            transition: all 0.3s;
        }
        
        .search-form input:focus {
            outline: none;
            border-color: #667eea;
        }
        
        .search-form button {
            padding: 15px 30px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            cursor: pointer;
            transition: transform 0.2s;
        }
        
        .search-form button:hover {
            transform: translateY(-2px);
        }
        
        .results {
            padding: 30px;
        }
        
        .business-card {
            background: white;
            border: 1px solid #e0e0e0;
            border-radius: 15px;
            padding: 20px;
            margin-bottom: 20px;
            transition: all 0.3s;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        
        .business-card:hover {
            transform: translateY(-3px);
            box-shadow: 0 5px 20px rgba(0,0,0,0.15);
        }
        
        .business-name {
            font-size: 20px;
            font-weight: bold;
            color: #333;
            margin-bottom: 15px;
            border-right: 4px solid #667eea;
            padding-right: 15px;
        }
        
        .contact-info {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
            margin: 15px 0;
        }
        
        .contact-item {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 8px 15px;
            background: #f8f9fa;
            border-radius: 8px;
        }
        
        .contact-actions {
            display: flex;
            gap: 10px;
            margin-top: 15px;
            flex-wrap: wrap;
        }
        
        .btn-call {
            background: #28a745;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
        }
        
        .btn-email {
            background: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
        }
        
        .btn-website {
            background: #17a2b8;
            color: white;
            text-decoration: none;
            padding: 10px 20px;
            border-radius: 8px;
            display: inline-block;
            font-size: 14px;
        }
        
        .loading {
            text-align: center;
            padding: 50px;
            font-size: 18px;
            color: #667eea;
        }
        
        .stats {
            background: #e9ecef;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 20px;
            text-align: center;
        }
        
        @media (max-width: 768px) {
            .search-form {
                flex-direction: column;
            }
            
            .contact-info {
                flex-direction: column;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🔍 سیستم هوشمند جستجوی کسب و کارها</h1>
            <p>جستجو، استخراج اطلاعات تماس و برقراری ارتباط مستقیم</p>
        </div>
        
        <div class="search-box">
            <form class="search-form" id="searchForm">
                <input type="text" id="topic" placeholder="موضوع مورد نظر (مثال: آرایشگاه، تعمیرات موبایل، رستوران)" required>
                <input type="text" id="location" placeholder="موقعیت (اختیاری - مثال: تهران)">
                <button type="submit">🔍 جستجوی کسب و کارها</button>
            </form>
        </div>
        
        <div class="results" id="results">
            <div class="stats" id="stats" style="display: none;"></div>
            <div id="businessList"></div>
        </div>
    </div>
    
    <script>
        document.getElementById('searchForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            
            const topic = document.getElementById('topic').value;
            const location = document.getElementById('location').value;
            const businessList = document.getElementById('businessList');
            const statsDiv = document.getElementById('stats');
            
            businessList.innerHTML = '<div class="loading">⏳ در حال جستجو و استخراج اطلاعات...<br>این فرآیند ممکن است چند لحظه طول بکشد</div>';
            statsDiv.style.display = 'none';
            
            try {
                const response = await fetch('/api/search', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({topic, location})
                });
                
                const data = await response.json();
                
                if (data.success && data.businesses.length > 0) {
                    statsDiv.style.display = 'block';
                    statsDiv.innerHTML = `✅ ${data.businesses.length} کسب و کار پیدا شد | 📊 ${data.saved} مورد جدید ذخیره شد`;
                    
                    businessList.innerHTML = data.businesses.map(biz => `
                        <div class="business-card">
                            <div class="business-name">🏢 ${biz.name || 'نامشخص'}</div>
                            <div class="contact-info">
                                ${biz.phone ? `
                                    <div class="contact-item">
                                        📞 ${biz.phone}
                                    </div>
                                ` : ''}
                                ${biz.email ? `
                                    <div class="contact-item">
                                        ✉️ ${biz.email}
                                    </div>
                                ` : ''}
                                ${biz.website ? `
                                    <div class="contact-item">
                                        🌐 ${biz.website.substring(0, 50)}...
                                    </div>
                                ` : ''}
                            </div>
                            <div class="contact-actions">
                                ${biz.phone ? `
                                    <button class="btn-call" onclick="makeCall('${biz.phone}')">
                                        📞 تماس مستقیم
                                    </button>
                                ` : ''}
                                ${biz.email ? `
                                    <button class="btn-email" onclick="sendEmail('${biz.email}', '${biz.name}')">
                                        ✉️ ارسال ایمیل
                                    </button>
                                ` : ''}
                                ${biz.website ? `
                                    <a href="${biz.website}" target="_blank" class="btn-website">
                                        🌐 بازدید سایت
                                    </a>
                                ` : ''}
                            </div>
                        </div>
                    `).join('');
                } else {
                    businessList.innerHTML = '<div class="stats">⚠️ هیچ نتیجه‌ای یافت نشد. لطفا عبارت دیگری جستجو کنید.</div>';
                }
            } catch (error) {
                businessList.innerHTML = '<div class="stats">❌ خطا در ارتباط با سرور</div>';
            }
        });
        
        function makeCall(phone) {
            window.location.href = `tel:${phone}`;
        }
        
        function sendEmail(email, name) {
            window.location.href = `mailto:${email}?subject=استعلام همکاری&body=سلام ${name}،%0D%0A%0D%0Aما از طریق جستجوی آنلاین با کسب و کار شما آشنا شدیم...%0D%0A%0D%0Aبا احترام`;
        }
        
        // بارگذاری خودکار آمار
        async function loadStats() {
            const response = await fetch('/api/stats');
            const data = await response.json();
            if (data.total > 0) {
                document.getElementById('stats').style.display = 'block';
                document.getElementById('stats').innerHTML = `📊 آمار کلی: ${data.total} کسب و کار در دیتابیس | ${data.contacted} تماس گرفته شده`;
            }
        }
        
        loadStats();
    </script>
</body>
</html>
'''

@app.route('/')
def index():
    return HTML_TEMPLATE

@app.route('/api/search', methods=['POST'])
def search():
    data = request.json
    topic = data.get('topic', '')
    location = data.get('location', '')
    
    # جستجوی کسب و کارها
    businesses = extractor.search_businesses(topic, location)
    
    # ذخیره در دیتابیس
    saved = db.save_businesses(businesses, topic)
    
    return jsonify({
        'success': True,
        'businesses': businesses[:50],  # حداکثر 50 نتیجه
        'saved': saved,
        'total_found': len(businesses)
    })

@app.route('/api/businesses', methods=['GET'])
def get_businesses():
    topic = request.args.get('topic')
    businesses = db.get_all_businesses(topic)
    return jsonify({'businesses': businesses})

@app.route('/api/contact/<int:business_id>', methods=['POST'])
def update_contact(business_id):
    data = request.json
    db.update_contact_status(business_id, True, data.get('notes', ''))
    return jsonify({'success': True})

@app.route('/api/export', methods=['GET'])
def export():
    filename = db.export_to_excel()
    return jsonify({'file': filename})

@app.route('/api/stats', methods=['GET'])
def stats():
    businesses = db.get_all_businesses()
    contacted = sum(1 for b in businesses if b.get('contacted'))
    return jsonify({
        'total': len(businesses),
        'contacted': contacted
    })

# ==================== بخش چهارم: اجرای اصلی ====================

def main():
    print("=" * 60)
    print("🚀 سیستم هوشمند جستجوی کسب و کارها")
    print("=" * 60)
    print("\n✅ دیتابیس آماده است")
    print("✅ کرالر وب فعال شد")
    print("🌐 وب سرور در حال اجرا...")
    print("\n👉 برای استفاده، مرورگر خود را باز کنید و به آدرس زیر بروید:")
    print("   http://localhost:5000")
    print("\n⚠️  برای توقف سرور، کلید Ctrl+C را بزنید")
    print("=" * 60)
    
    app.run(debug=True, host='0.0.0.0', port=5000)

if __name__ == '__main__':
    main()
