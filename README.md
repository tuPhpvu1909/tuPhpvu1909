import tkinter as tk
from collections import deque, Counter
import random
import json

# Khởi tạo các ô với giá trị cố định và tỷ lệ trúng thưởng
slots = [
    {"name": "Trâu", "value": 5, "probability": 0.12},
    {"name": "Trâu", "value": 6, "probability": 0.10},
    {"name": "Trâu", "value": 7, "probability": 0.08},
    {"name": "Tăng", "value": 8, "probability": 0.07},
    {"name": "Tăng", "value": 10, "probability": 0.06},
    {"name": "Tăng", "value": 11, "probability": 0.05},
    {"name": "Heo", "value": 17, "probability": 0.04},
    {"name": "Heo", "value": 19, "probability": 0.03},
    {"name": "Heo", "value": 22, "probability": 0.025},
    {"name": "Khỉ", "value": 31, "probability": 0.02},
    {"name": "Khỉ", "value": 35, "probability": 0.015},
    {"name": "Khỉ", "value": 40, "probability": 0.01},
    {"name": "Sao", "value": None, "probability": 0.005},
    {"name": "Chum", "value": None, "probability": 0.002},
]

# Mapping từ viết tắt sang tên đầy đủ
abbreviation_map = {
    "tr": "Trâu",
    "t": "Tăng",
    "h": "Heo",
    "k": "Khỉ",
    "sao": "Sao",
    "chum": "Chum"
}

# Lưu trữ kết quả
recent_results = deque(maxlen=20)
selected_abbreviation = None
selected_value = None

def create_keyboard():
    window = tk.Tk()
    window.title("Bàn phím ảo nhập kết quả")

    # Biến chọn phương pháp dự đoán
    selected_method = tk.StringVar(value="weighted")

    # Nút cho từng giá trị và tên viết tắt
    buttons_frame = tk.Frame(window)
    buttons_frame.pack()

    for name, values in abbreviation_map.items():
        if name in ["tr", "t", "h", "k"]:
            for value in [slot["value"] for slot in slots if slot["name"] == abbreviation_map[name]]:
                button = tk.Button(buttons_frame, text=f"{name.upper()}-{value}", command=lambda n=name, v=value: on_button_click(n, v))
                button.pack(side=tk.LEFT)
        else:
            button = tk.Button(buttons_frame, text=f"{name.upper()}", command=lambda n=name: on_button_click(n, None))
            button.pack(side=tk.LEFT)

    # Hiển thị kết quả phân tích và quay ô
    analyze_button = tk.Button(window, text="Phân tích kết quả & Quay ô", command=lambda: analyze_and_spin(selected_method))
    analyze_button.pack()

    # Nút xóa kết quả
    clear_button = tk.Button(window, text="Xóa kết quả", command=clear_results)
    clear_button.pack()

    # Nút hoàn tác (Undo) kết quả cuối cùng
    undo_button = tk.Button(window, text="Undo", command=undo_result)
    undo_button.pack()

    # Nút xem kết quả đã nhập
    view_button = tk.Button(window, text="Xem kết quả đã nhập", command=view_results)
    view_button.pack()

    # Chọn phương pháp dự đoán
    method_frame = tk.Frame(window)
    method_frame.pack()

    tk.Radiobutton(method_frame, text="Dựa trên trọng số", variable=selected_method, value="weighted").pack(side=tk.LEFT)
    tk.Radiobutton(method_frame, text="Trung bình di chuyển", variable=selected_method, value="moving_average").pack(side=tk.LEFT)

    # Label hiển thị kết quả
    global result_label
    result_label = tk.Label(window, text="")
    result_label.pack()

    window.mainloop()

# Hàm xử lý khi nhấn nút
def on_button_click(name, value):
    global selected_abbreviation, selected_value
    selected_abbreviation = name
    selected_value = value
    recent_results.append({"name": abbreviation_map[name], "value": value})
    display_results()

# Hàm hiển thị kết quả kèm thứ tự
def display_results():
    result_text = "\n".join([f"{i+1}. {result['name']} - {result['value']}" for i, result in enumerate(recent_results)])
    result_label.config(text=result_text)

# Phương pháp dự đoán dựa trên trọng số (Weighted Prediction)
def weighted_prediction():
    counts = Counter(result['name'] for result in recent_results)
    weights = []

    for slot in slots:
        count = counts.get(slot['name'], 0)
        weight = slot['value'] if slot['value'] else 1  # Trọng số dựa trên giá trị hoặc 1 nếu là "Sao" hoặc "Chum"
        weight /= (count + 1)
        weights.append(weight)

    predicted_slot = random.choices(slots, weights=weights, k=1)[0]
    return predicted_slot['name']

# Phương pháp dự đoán theo trung bình di chuyển (Moving Average Prediction)
def moving_average_prediction(window_size=3):
    if len(recent_results) < window_size:
        window_size = len(recent_results)
    recent_slots = list(recent_results)[-window_size:]
    counts = Counter(result['name'] for result in recent_slots)
    return counts.most_common(1)[0][0]

# Phân tích tần suất và quay ô
def analyze_and_spin(selected_method):
    counts = Counter(result['name'] for result in recent_results)
    analysis_text = "Tần suất xuất hiện của các ô gần đây:\n"
    for name, count in counts.items():
        analysis_text += f"{name}: {count} lần\n"
    
    # Chọn phương pháp dự đoán
    if selected_method.get() == "weighted":
        predicted_name = weighted_prediction()
        analysis_text += f"Dự đoán tiếp theo (dựa trên trọng số): {predicted_name}\n\n"
    else:
        predicted_name = moving_average_prediction()
        analysis_text += f"Dự đoán tiếp theo (theo trung bình di chuyển): {predicted_name}\n\n"

    # Quay ô ngẫu nhiên
    chosen_slot = random.choices(slots, weights=[slot['probability'] for slot in slots])[0]
    spin_result = f"Kết quả quay ô: {chosen_slot['name']} với giá trị {chosen_slot['value']}\n"

    result_label.config(text=analysis_text + spin_result)

# Hàm xóa tất cả các kết quả đã nhập
def clear_results():
    recent_results.clear()
    display_results()

# Hàm hoàn tác (Undo) kết quả cuối cùng
def undo_result():
    if recent_results:
        recent_results.pop()
    display_results()

# Hàm xem tất cả các kết quả đã nhập
def view_results():
    display_results()

# Kiểm tra nếu đây là lần đầu chạy, nhập 20 kết quả
try:
    with open('results_cache.json', 'r') as file:
        recent_results = deque(json.load(file), maxlen=20)
except FileNotFoundError:
    pass  # Không có file thì để người dùng nhập từ bàn phím ảo

# Tạo bàn phím ảo và hiển thị kết quả
create_keyboard()

# Lưu lại các kết quả vào file để sử dụng trong lần chạy sau
with open('results_cache.json', 'w') as file:
    json.dump(list(recent_results), file)
