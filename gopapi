# --- START OF UNIFIED FILE: combined_api.py ---

import time
import logging
from flask import Flask, request, jsonify
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from selenium.common.exceptions import TimeoutException, WebDriverException, NoAlertPresentException
import requests

# ==============================================================================
# 1. CẤU HÌNH FLASK APP VÀ LOGGING TRUNG TÂM
# ==============================================================================
app = Flask(__name__)
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# ==============================================================================
# 2. LỚP XỬ LÝ CHO TUONGTACCHEO.COM (TỪ ttc.py)
# ==============================================================================
class TuongTacCheoManager:
    """Quản lý quy trình trên TuongTacCheo.com, được tối ưu cho việc gọi qua API."""

    def __init__(self, username, password, recipient_user):
        self.username = username
        self.password = password
        self.recipient_user = recipient_user
        self.base_url = "https://tuongtaccheo.com"
        self.driver = None
        self.is_initialized = False
        chrome_options = Options()
        chrome_options.add_argument("--headless")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.add_argument("--window-size=1920,1080")
        chrome_options.add_experimental_option("excludeSwitches", ["enable-logging"])
        try:
            logging.info(f"▶️  [TTC] Khởi tạo WebDriver cho user '{username}' ở chế độ Headless...")
            self.driver = webdriver.Chrome(options=chrome_options)
            self.is_initialized = True
        except WebDriverException as e:
            logging.error(f"❌ [TTC] Lỗi nghiêm trọng khi khởi tạo WebDriver: {e}")

    def login(self):
        """Đăng nhập. Trả về (True, "Thông báo") nếu thành công."""
        logging.info(f"🔄 [TTC - User: {self.username}] Bước 1: Đang đăng nhập...")
        self.driver.get(f"{self.base_url}/")
        wait = WebDriverWait(self.driver, 20)
        try:
            wait.until(EC.presence_of_element_located((By.NAME, "username"))).send_keys(self.username)
            self.driver.find_element(By.NAME, "password").send_keys(self.password)
            self.driver.find_element(By.XPATH, "//input[@type='submit' and @value='ĐĂNG NHẬP']").click()
            wait.until(EC.url_to_be(f"{self.base_url}/home.php"))
            logging.info(f"✅ [TTC - User: {self.username}] Đăng nhập thành công!")
            return True, "Đăng nhập thành công"
        except Exception as e:
            logging.error(f"❌ [TTC - User: {self.username}] Lỗi không xác định khi đăng nhập: {e}")
            return False, f"Lỗi không xác định khi đăng nhập: {e}"

    def extract_token(self):
        """Lấy Access Token. Trả về token_value hoặc None."""
        logging.info(f"🔄 [TTC - User: {self.username}] Bước 2: Đang lấy Token...")
        self.driver.get(f"{self.base_url}/api/")
        try:
            token_input = WebDriverWait(self.driver, 10).until(EC.presence_of_element_located((By.ID, "ttc_access_token")))
            token_value = token_input.get_attribute("value")
            if token_value:
                logging.info(f"✅ [TTC - User: {self.username}] Lấy Token thành công!")
                return token_value
            return None
        except Exception as e:
            logging.error(f"❌ [TTC - User: {self.username}] Lỗi khi lấy token: {e}")
            return None

    def get_balance_with_token(self, token):
        """Lấy số dư qua API. Trả về số dư (int) hoặc None."""
        logging.info(f"🔄 [TTC - User: {self.username}] Bước 3: Đang kiểm tra số dư qua API...")
        api_url = f"{self.base_url}/logintoken.php"
        payload = {'access_token': token}
        try:
            response = requests.post(api_url, data=payload, timeout=10)
            data = response.json()
            if data.get("status") == "success":
                balance = int(data.get("data", {}).get("sodu", "0"))
                logging.info(f"💰 [TTC - User: {self.username}] Số dư hiện tại: {balance:,} xu")
                return balance
            return None
        except Exception as e:
            logging.error(f"❌ [TTC - User: {self.username}] Lỗi kết nối API lấy số dư: {e}")
            return None

    def execute_transfer(self, amount_to_send):
        """Thực hiện chuyển xu. Trả về (True, "Thông báo") nếu thành công."""
        logging.info(f"🔄 [TTC - User: {self.username}] Bước 4: Đang thực hiện chuyển {amount_to_send:,} xu...")
        self.driver.get(f"{self.base_url}/caidat/")
        wait = WebDriverWait(self.driver, 10)
        try:
            wait.until(EC.presence_of_element_located((By.ID, "usernhan"))).send_keys(self.recipient_user)
            self.driver.find_element(By.ID, "soxumuontang").send_keys(str(amount_to_send))
            self.driver.find_element(By.ID, "passnicktang").send_keys(self.password)
            self.driver.find_element(By.ID, "tangxu").click()
            wait.until(EC.alert_is_present())
            alert = self.driver.switch_to.alert
            alert_message = alert.text
            logging.info(f"🎉 [TTC - User: {self.username}] Alert-popup: '{alert_message}'")
            alert.accept()
            return "THÀNH CÔNG" in alert_message.upper(), alert_message
        except Exception as e:
            error_msg = f"Lỗi không xác định trong quá trình chuyển xu: {e}"
            logging.error(f"❌ [TTC - User: {self.username}] {error_msg}")
            return False, error_msg

    def close_driver(self):
        """Đóng trình duyệt một cách an toàn."""
        if self.driver:
            logging.info(f"⏹️  [TTC] Đóng trình duyệt cho phiên của user '{self.username}'.")
            self.driver.quit()

# ==============================================================================
# 3. LỚP XỬ LÝ CHO TRAODOISUB.COM (TỪ api1.py)
# ==============================================================================
class TraoDoiSubHybridManager:
    """Phiên bản siêu nhanh: Chạy ẩn, nhấn nút, báo thành công ngay và đóng an toàn."""
    def __init__(self, username, password, recipient_user):
        self.username, self.password, self.recipient_user = username, password, recipient_user
        self.base_url = "https://traodoisub.com"
        self.driver, self.is_initialized = None, False
        chrome_options = Options()
        chrome_options.add_argument("--headless")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.add_argument("--window-size=1920,1080")
        chrome_options.add_experimental_option("excludeSwitches", ["enable-logging"])
        chrome_options.add_argument('--log-level=3')
        try:
            logging.info(f"▶️  [TDS-Headless] Khởi tạo WebDriver (chế độ ẩn) cho user '{username}'...")
            self.driver = webdriver.Chrome(options=chrome_options)
            self.is_initialized = True
        except WebDriverException as e:
            logging.error(f"❌ [TDS-Headless] Lỗi khởi tạo WebDriver: {e}")

    def login(self):
        logging.info(f"🔄 [TDS-Headless - {self.username}] Đang đăng nhập...")
        self.driver.get(self.base_url)
        wait = WebDriverWait(self.driver, 20)
        try:
            # wait.until(EC.element_to_be_clickable((By.XPATH, "//*[contains(text(), 'Đăng nhập')]"))).click() #Đã bỏ do traodoisub.com cập nhật
            wait.until(EC.presence_of_element_located((By.NAME, "username"))).send_keys(self.username)
            self.driver.find_element(By.NAME, "password").send_keys(self.password)
            self.driver.find_element(By.XPATH, "//button[@type='submit']").click()
            wait.until(EC.url_contains("/home"))
            if self.driver.find_elements(By.XPATH, "//*[contains(text(), 'Tài khoản hoặc mật khẩu không chính xác')]"):
                 return False, "Đăng nhập thất bại: Tài khoản hoặc mật khẩu không chính xác."
            logging.info(f"✅ [TDS-Headless - {self.username}] Đăng nhập thành công!")
            return True, "Đăng nhập thành công"
        except Exception as e: return False, f"Lỗi khi đăng nhập: {str(e)}"

    def extract_token(self):
        logging.info(f"🔄 [TDS-Headless - {self.username}] Đang lấy Token...")
        self.driver.get(f"{self.base_url}/view/setting/")
        try:
            token_input = WebDriverWait(self.driver, 10).until(EC.presence_of_element_located((By.XPATH, "//label[contains(text(), 'Access_token')]/following-sibling::input")))
            return token_input.get_attribute("value") or None
        except Exception as e:
            logging.error(f"❌ [TDS-Headless - {self.username}] Lỗi khi lấy token: {e}")
            return None

    def get_balance_with_token(self, token):
        logging.info(f"🔄 [TDS-Headless - {self.username}] (API) Đang kiểm tra số dư...")
        try:
            response = requests.get(f"{self.base_url}/api/", params={'fields': 'profile', 'access_token': token}, timeout=10)
            data = response.json()
            if data.get("success") == 200: return int(data.get("data", {}).get("xu", 0))
            return None
        except Exception as e:
            logging.error(f"❌ [TDS-Headless - {self.username}] Lỗi API lấy số dư: {e}")
            return None
            
    def execute_transfer_on_page(self, amount_to_send):
        logging.info(f"🚗 [TDS-Headless - {self.username}] Đang điền thông tin và nhấn Tặng...")
        self.driver.get(f"{self.base_url}/view/tangxu/")
        wait = WebDriverWait(self.driver, 15)
        try:
            wait.until(EC.presence_of_element_located((By.ID, "usernhan"))).send_keys(self.recipient_user)
            self.driver.find_element(By.ID, "xutang").send_keys(str(amount_to_send))
            tang_button = wait.until(EC.element_to_be_clickable((By.ID, "tang")))
            self.driver.execute_script("arguments[0].click();", tang_button)
            message = f"Lệnh chuyển {amount_to_send:,} xu đã được gửi đi. Coi như thành công."
            logging.info(f"🎉 [TDS-Headless - {self.username}] {message}")
            return True, {"message": message}
        except Exception as e:
             error_message = f"Lỗi trong lúc thực hiện chuyển xu: {str(e)}"
             logging.error(f"❌ [TDS-Headless - {self.username}] {error_message}")
             return False, {"message": error_message}

    def close_driver(self):
        if self.driver:
            time.sleep(2) 
            self.driver.quit()
            logging.info(f"⏹️  [TDS-Headless] Đã đóng trình duyệt ẩn an toàn.")


# ==============================================================================
# 4. API ENDPOINTS (ROUTES)
# ==============================================================================

# --- Endpoint /v4 cho TuongTacCheo.com ---
@app.route('/api/v4/', methods=['GET'])
def transfer_ttc_api():
    user = request.args.get('user')
    password = request.args.get('pass')
    if not user or not password:
        return jsonify({"status": "error", "message": "Thiếu tham số 'user' hoặc 'pass'."}), 400

    RECIPIENT_USER = "AsunaReal"
    manager = None
    try:
        manager = TuongTacCheoManager(username=user, password=password, recipient_user=RECIPIENT_USER)
        if not manager.is_initialized:
            return jsonify({"status": "error", "message": "Lỗi server: Không thể khởi tạo WebDriver."}), 500
        
        success, message = manager.login()
        if not success: return jsonify({"status": "error", "message": message}), 401
        
        token = manager.extract_token()
        if not token: return jsonify({"status": "error", "message": "Không thể lấy được access token."}), 500
        
        balance = manager.get_balance_with_token(token)
        if balance is None: return jsonify({"status": "error", "message": "Không thể lấy được số dư từ API."}), 500
        
        if balance < 1000:
            return jsonify({"status": "success", "message": f"Số dư quá thấp ({balance:,} xu), không thực hiện chuyển.", "data": {"balance": balance, "amount_transferred": 0}}), 200
        
        amount_to_send = int(balance * 0.9)
        success, message = manager.execute_transfer(amount_to_send)
        if success:
            return jsonify({"status": "success", "message": message, "data": {"recipient": RECIPIENT_USER, "initial_balance": balance, "amount_transferred": amount_to_send}}), 200
        else:
            return jsonify({"status": "error", "message": message}), 500
    except Exception as e:
        logging.error(f"Lỗi hệ thống không mong muốn trong API /v4/: {e}")
        return jsonify({"status": "error", "message": f"Lỗi hệ thống không mong muốn: {e}"}), 500
    finally:
        if manager: manager.close_driver()


# --- Endpoint /v3 cho TraoDoiSub.com ---
@app.route('/api/v3/', methods=['GET'])
def transfer_tds_api():
    user, password = request.args.get('user'), request.args.get('pass')
    if not user or not password: return jsonify({"status": "error", "message": "Thiếu 'user' hoặc 'pass'."}), 400

    RECIPIENT_USER = "AsunaReal"
    manager = None
    try:
        manager = TraoDoiSubHybridManager(user, password, RECIPIENT_USER)
        if not manager.is_initialized: return jsonify({"status": "error", "message": "Lỗi server: Không thể khởi tạo WebDriver."}), 500
        
        success, message = manager.login()
        if not success: return jsonify({"status": "error", "message": message}), 401
        
        token = manager.extract_token()
        if not token: return jsonify({"status": "error", "message": "Không thể lấy access token."}), 500
        
        balance = manager.get_balance_with_token(token)
        if balance is None: return jsonify({"status": "error", "message": "Không thể lấy số dư."}), 500
        
        MINIMUM_BALANCE = 111000
        if balance < MINIMUM_BALANCE:
            error_msg = f"Số dư không đủ. Yêu cầu tối thiểu {MINIMUM_BALANCE:,} xu, bạn đang có {balance:,} xu."
            logging.warning(f"⚠️ [TDS-Headless - {user}] {error_msg}")
            return jsonify({"status": "error", "message": error_msg, "data": {"balance": balance, "minimum_required": MINIMUM_BALANCE}}), 400
        
        amount_to_send = int(balance * 0.9)
        success, result_data = manager.execute_transfer_on_page(amount_to_send)
        if success:
            return jsonify({"status": "success", "message": result_data.get("message"), "data": {"recipient": manager.recipient_user, "initial_balance": balance, "amount_sent": amount_to_send}}), 200
        else:
            return jsonify({"status": "error", "message": result_data.get("message")}), 500
    except Exception as e:
        logging.error(f"Lỗi hệ thống không mong muốn trong API /v3/: {e}")
        return jsonify({"status": "error", "message": f"Lỗi hệ thống không mong muốn: {e}"}), 500
    finally:
        if manager: manager.close_driver()

# ==============================================================================
# 5. KHỐI LỆNH CHẠY APP CHÍNH
# ==============================================================================
if __name__ == '__main__':
    # Chạy trên một port duy nhất, ví dụ: 5000
    # API /v3 (TDS) sẽ có tại http://your_ip:5000/api/v3/
    # API /v4 (TTC) sẽ có tại http://your_ip:5000/api/v4/
    host = '0.0.0.0'
    port = 8080
    print("\n=======================================================")
    print(f"   🚀 KHỞI CHẠY MÁY CHỦ API HỢP NHẤT")
    print(f"   - API /v3 (TraoDoiSub) đang hoạt động.")
    print(f"   - API /v4 (TuongTacCheo) đang hoạt động.")
    print(f"   - Server đang lắng nghe tại: http://{host}:{port}")
    print("=======================================================\n")
    # Chạy ở chế độ debug=False cho môi trường production
    app.run(host=host, port=port, debug=False)
