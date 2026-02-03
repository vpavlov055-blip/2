import urllib.request
import json
import time
import os
import tempfile

TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "")
URL = f"https://api.telegram.org/bot{TOKEN}/"

def make_request(method, data=None):
    try:
        if data:
            data = json.dumps(data).encode()
            req = urllib.request.Request(URL + method, data=data, 
                                       headers={'Content-Type': 'application/json'})
        else:
            req = urllib.request.Request(URL + method)
        
        with urllib.request.urlopen(req, timeout=10) as response:
            return json.load(response)
    except:
        return {"ok": False}

def send_message(chat_id, text, reply_markup=None):
    data = {"chat_id": chat_id, "text": text}
    if reply_markup:
        data["reply_markup"] = reply_markup
    make_request("sendMessage", data)

def send_document(chat_id, filepath):
    try:
        import requests
        files = {'document': open(filepath, 'rb')}
        data = {'chat_id': chat_id}
        response = requests.post(
            f'https://api.telegram.org/bot{TOKEN}/sendDocument',
            files=files,
            data=data
        )
        files['document'].close()
        return response.status_code == 200
    except:
        try:
            import mimetypes
            boundary = '----WebKitFormBoundary' + ''.join([str(i) for i in range(10)])
            headers = {'Content-Type': f'multipart/form-data; boundary={boundary}'}
            with open(filepath, 'rb') as f:
                file_content = f.read()
            filename = os.path.basename(filepath)
            body = (
                f'--{boundary}\r\n'
                f'Content-Disposition: form-data; name="chat_id"\r\n\r\n'
                f'{chat_id}\r\n'
                f'--{boundary}\r\n'
                f'Content-Disposition: form-data; name="document"; filename="{filename}"\r\n'
                f'Content-Type: application/octet-stream\r\n\r\n'
            ).encode() + file_content + f'\r\n--{boundary}--\r\n'.encode()
            req = urllib.request.Request(
                f'https://api.telegram.org/bot{TOKEN}/sendDocument',
                data=body,
                headers=headers
            )
            with urllib.request.urlopen(req) as response:
                result = json.load(response)
                return result.get('ok', False)
        except:
            return False

def get_conversion_keyboard():
    return {
        "inline_keyboard": [
            [
                {"text": "TXT → PDF", "callback_data": "txt_pdf"},
                {"text": "JPG → PNG", "callback_data": "jpg_png"}
            ],
            [
                {"text": "DOCX → TXT", "callback_data": "docx_txt"},
                {"text": "PNG → JPG", "callback_data": "png_jpg"}
            ]
        ]
    }

def get_back_button():
    return {
        "inline_keyboard": [
            [{"text": "Назад", "callback_data": "back"}]
        ]
    }

def download_file(file_id, filename):
    file_info = make_request(f"getFile?file_id={file_id}")
    if file_info.get("ok"):
        file_path = file_info["result"]["file_path"]
        file_url = f"https://api.telegram.org/file/bot{TOKEN}/{file_path}"
        urllib.request.urlretrieve(file_url, filename)
        return True
    return False

def convert_txt_to_pdf_fpdf(input_path, output_path):
    try:
        from fpdf import FPDF
        pdf = FPDF()
        pdf.add_page()
        pdf.add_font('DejaVu', '', 'DejaVuSans.ttf', uni=True)
        pdf.set_font('DejaVu', '', 12)
        
        with open(input_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()
        
        for line in lines:
            pdf.cell(200, 10, txt=line.strip(), ln=1)
        
        pdf.output(output_path)
        return True
    except:
        try:
            with open(input_path, 'r', encoding='utf-8') as f:
                content = f.read()
            with open(output_path, 'w', encoding='utf-8') as f:
                f.write(f"PDF версия:\n\n{content}")
            return True
        except:
            return False

def convert_image_pil(input_path, output_path):
    try:
        from PIL import Image
        with Image.open(input_path) as img:
            img.save(output_path)
        return True
    except:
        return False

def convert_docx_to_txt_python_docx(input_path, output_path):
    try:
        from docx import Document
        doc = Document(input_path)
        text = '\n'.join([paragraph.text for paragraph in doc.paragraphs])
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(text)
        return True
    except:
        return False

print("Бот запущен...")
last_id = 0
user_states = {}

while True:
    try:
        updates = make_request(f"getUpdates?offset={last_id+1}&timeout=60")
        
        if updates.get("result"):
            for update in updates["result"]:
                last_id = update["update_id"]
                
                if "callback_query" in update:
                    query = update["callback_query"]
                    chat_id = query["message"]["chat"]["id"]
                    data = query["data"]
                    message_id = query["message"]["message_id"]
                    
                    make_request("editMessageReplyMarkup", {
                        "chat_id": chat_id,
                        "message_id": message_id,
                        "reply_markup": {"inline_keyboard": []}
                    })
                    
                    if data == "txt_pdf":
                        send_message(chat_id, "TXT → PDF\n\nОтправьте текстовый файл (.txt)", get_back_button())
                        user_states[chat_id] = "convert_txt_pdf"
                    
                    elif data == "jpg_png":
                        send_message(chat_id, "JPG → PNG\n\nОтправьте JPG изображение", get_back_button())
                        user_states[chat_id] = "convert_jpg_png"
                    
                    elif data == "docx_txt":
                        send_message(chat_id, "DOCX → TXT\n\nОтправьте документ Word (.docx)", get_back_button())
                        user_states[chat_id] = "convert_docx_txt"
                    
                    elif data == "png_jpg":
                        send_message(chat_id, "PNG → JPG\n\nОтправьте PNG изображение", get_back_button())
                        user_states[chat_id] = "convert_png_jpg"
                    
                    elif data == "back":
                        send_message(chat_id, "Выберите тип конвертации:", get_conversion_keyboard())
                        user_states[chat_id] = "main"
                    
                    make_request("answerCallbackQuery", {"callback_query_id": query["id"]})
                
                elif "message" in update:
                    msg = update["message"]
                    chat_id = msg.get("chat", {}).get("id")
                    text = msg.get("text", "")
                    first_name = msg.get("chat", {}).get("first_name", "")
                    
                    if not chat_id:
                        continue
                    
                    if text == "/start":
                        welcome_text = f"Привет, {first_name}!\n\nЯ бот для конвертации файлов\n\nВыберите действие:"
                        send_message(chat_id, welcome_text, get_conversion_keyboard())
                        user_states[chat_id] = "main"
                    
                    elif "document" in msg:
                        doc = msg["document"]
                        filename = doc.get("file_name", "file")
                        
                        send_message(chat_id, f"Получен файл: {filename}")
                        
                        with tempfile.TemporaryDirectory() as tmp_dir:
                            input_path = os.path.join(tmp_dir, filename)
                            
                            if download_file(doc["file_id"], input_path):
                                state = user_states.get(chat_id, "")
                                base_name = os.path.splitext(filename)[0]
                                result = False
                                
                                if state == "convert_txt_pdf" and filename.lower().endswith('.txt'):
                                    output_path = os.path.join(tmp_dir, base_name + ".pdf")
                                    result = convert_txt_to_pdf_fpdf(input_path, output_path)
                                
                                elif state == "convert_jpg_png" and filename.lower().endswith(('.jpg', '.jpeg')):
                                    output_path = os.path.join(tmp_dir, base_name + ".png")
                                    result = convert_image_pil(input_path, output_path)
                                
                                elif state == "convert_docx_txt" and filename.lower().endswith('.docx'):
                                    output_path = os.path.join(tmp_dir, base_name + ".txt")
                                    result = convert_docx_to_txt_python_docx(input_path, output_path)
                                
                                elif state == "convert_png_jpg" and filename.lower().endswith('.png'):
                                    output_path = os.path.join(tmp_dir, base_name + ".jpg")
                                    result = convert_image_pil(input_path, output_path)
                                
                                if result:
                                    if send_document(chat_id, output_path):
                                        send_message(chat_id, "Конвертация завершена")
                                    else:
                                        send_message(chat_id, "Ошибка отправки файла")
                                else:
                                    send_message(chat_id, "Ошибка конвертации")
                                
                                send_message(chat_id, "Выберите следующий тип конвертации:", get_conversion_keyboard())
                                user_states[chat_id] = "main"
                            
                            else:
                                send_message(chat_id, "Ошибка загрузки", get_conversion_keyboard())
        
    except Exception as e:
        print(f"Ошибка: {e}")
        time.sleep(5)
