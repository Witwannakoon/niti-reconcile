import streamlit as st
import pdfplumber
import pandas as pd

# ตั้งค่าหน้าเว็บ
st.set_page_config(page_title="Niti Matching System", layout="wide")

st.title("🏢 ระบบตรวจจับคู่เอกสารนิติบุคคล")
st.write("เปรียบเทียบรายงานใบนำส่งเงิน กับ สลิปหลักฐานการโอน")

# ส่วนที่ 1: Sidebar สำหรับอัปโหลดไฟล์
with st.sidebar:
    st.header("📁 อัปโหลดเอกสาร")
    report_file = st.file_uploader("1. อัปโหลดรายงานนิติ (PDF)", type="pdf")
    slip_files = st.file_uploader("2. อัปโหลดสลิป/หลักฐาน (เลือกได้หลายไฟล์)", type=["pdf", "png", "jpg"], accept_multiple_files=True)

# ฟังก์ชันสำหรับดึงข้อมูลจากตารางใน PDF ของนิติ
def extract_data_from_pdf(file):
    all_data = []
    with pdfplumber.open(file) as pdf:
        # วนลูปอ่านทุกหน้า (เผื่อรายงานมีหลายหน้า)
        for page in pdf.pages:
            table = page.extract_table()
            if table:
                # table[0] มักจะเป็นหัวข้อ (Header) เราจะเริ่มเอาข้อมูลที่แถวที่ 1 เป็นต้นไป
                for row in table[1:]:
                    # ตรวจสอบว่าแถวมีข้อมูลเลขที่ห้อง (Column index 1)
                    if row[1] and row[1] != "เลขที่ห้อง":
                        all_data.append({
                            "เลขที่ห้อง": row[1].strip(),
                            "ชื่อลูกค้า": row[2].replace('\n', ' ') if row[2] else "",
                            "ยอดเงินโอน": float(row[4].replace(',', '')) if row[4] else 0.0,
                            "สถานะ": row[5].strip() if row[5] else "",
                            "ค่าบริการ": float(row[9].replace(',', '')) if row[9] else 0.0
                        })
    return pd.DataFrame(all_data)

# ส่วนการประมวลผล
if report_file:
    try:
        df = extract_data_from_pdf(report_file)
        
        st.subheader("📋 ข้อมูลจากรายงานนิติบุคคล")
        
        # แสดงผลตารางที่อ่านได้
        st.dataframe(df, use_container_width=True)

        # ส่วนการจับคู่ (Matching Logic)
        st.divider()
        st.subheader("🔍 ผลการตรวจสอบความถูกต้อง")
        
        if slip_files:
            st.info(f"ระบบตรวจพบไฟล์หลักฐาน {len(slip_files)} ไฟล์")
            
            # (ในอนาคต: ตรงนี้จะใส่โค้ด AI สำหรับอ่านรูปสลิป)
            # ตอนนี้จำลองการ Check เบื้องต้นจากสถานะ "โอน" ในไฟล์
            df['ผลการตรวจสอบ'] = df['สถานะ'].apply(lambda x: "✅ พบหลักฐาน (ตามรายงาน)" if x == "โอน" else "⚠️ ต้องตรวจสอบเพิ่ม (เงินสด/ยกเลิก)")
        else:
            df['ผลการตรวจสอบ'] = "⏳ รอการอัปโหลดสลิปเพื่อเปรียบเทียบ"

        # แสดงตารางผลลัพธ์
        st.table(df[['เลขที่ห้อง', 'ชื่อลูกค้า', 'ยอดเงินโอน', 'ผลการตรวจสอบ']])

        # ส่วนสรุปค่าธรรมเนียม 5 บาท
        st.divider()
        total_service = df[df['ค่าบริการ'] > 0]['ค่าบริการ'].count()
        profit = total_service * 5
        
        c1, c2 = st.columns(2)
        c1.metric("จำนวนรายการค่าบริการ", f"{total_service} รายการ")
        c2.metric("กำไรส่วนต่างนิติ (5 THB/ราย)", f"{profit} บาท", delta="รายได้อื่นๆ")

    except Exception as e:
        st.error(f"เกิดข้อผิดพลาดในการอ่านไฟล์: {e}")
else:
    st.info("กรุณาอัปโหลดไฟล์รายงาน PDF ที่แถบด้านข้างเพื่อเริ่มการทำงาน")