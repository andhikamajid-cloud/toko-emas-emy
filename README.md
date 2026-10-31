# toko-emas-emy
import streamlit as st
import pandas as pd
from datetime import datetime

# Konfigurasi Halaman (Mobile-Friendly)
st.set_page_config(
    page_title="POS Toko Emas - Buyback & Margin 10%",
    page_icon="🪙",
    layout="centered"
)

# Custom Styling CSS untuk tampilan mobile yang elegan
st.markdown("""
    <style>
    .main {
        background-color: #F8F9FA;
    }
    .stButton>button {
        width: 100%;
        background-color: #1F497D;
        color: white;
        font-weight: bold;
        border-radius: 8px;
        padding: 10px;
    }
    </style>
""", unsafe_allow_html=True)

# Judul Aplikasi
st.title("🪙 POS Toko Emas & Taksiran")
st.caption(f"📅 Tanggal: {datetime.now().strftime('%d %B %Y')} | Target Margin: 10%")

# --- 1. PANEL PENGATURAN HARGA ACUAN 24K ---
with st.container():
    st.markdown("### ⚙️ Master Harga Acuan")
    harga_24k = st.number_input(
        "Harga Acuan Emas 24K Hari Ini (Rp / Gram):",
        min_value=100000,
        max_value=10000000,
        value=1400000,
        step=5000,
        format="%d"
    )

st.markdown("---")

# --- 2. TAB MENU UTAMA ---
tab1, tab2, tab3 = st.tabs(["🛒 Input Buyback", "🧮 Kalkulator Taksiran", "📊 Rekap Harian"])

# Inisialisasi Session State untuk menyimpan riwayat transaksi harian
if "transactions" not in st.session_state:
    st.session_state.transactions = []

# --- TAB 1: INPUT BUYBACK ---
with tab1:
    st.subheader("Catat Transaksi Pembelian (Buyback)")
    
    with st.form("form_buyback", clear_on_submit=True):
        nama_pelanggan = st.text_input("Nama Pelanggan / Nota")
        karat_pilihan = st.selectbox("Pilih Karat Barang", [f"{k} Karat ({k}K)" for k in range(24, 0, -1)])
        berat_gram = st.number_input("Berat Barang (Gram)", min_value=0.01, max_value=1000.0, value=5.0, step=0.1)
        
        k_num = int(karat_pilihan.split(" ")[0])
        kadar_pct = 0.99 if k_num == 24 else (0.875 if k_num == 22 else (0.75 if k_num == 18 else (0.70 if k_num == 17 else k_num / 24.0)))
            
        harga_pasaran_gram = harga_24k * kadar_pct
        harga_beli_maks_gram = harga_pasaran_gram * (1 - 0.10) # Margin 10%
        
        saran_total_beli = harga_beli_maks_gram * berat_gram
        total_bayar_aktual = st.number_input("Total Pembayaran ke Pelanggan (Rp)", min_value=0.0, value=float(round(saran_total_beli, -2)), step=1000.0)
        
        submitted = st.form_submit_button("Simpan Transaksi")
        
        if submitted:
            profit_transaksi = (harga_pasaran_gram * berat_gram) - total_bayar_aktual
            st.session_state.transactions.append({
                "Waktu": datetime.now().strftime("%H:%M"),
                "Pelanggan": nama_pelanggan if nama_pelanggan else "Umum",
                "Karat": karat_pilihan,
                "Berat (g)": berat_gram,
                "Total Bayar": total_bayar_aktual,
                "Estimasi Profit": profit_transaksi
            })
            st.success(f"Transaksi berhasil disimpan! Total Bayar: Rp {total_bayar_aktual:,.0f}")

    if st.session_state.transactions:
        st.markdown("#### Daftar Transaksi Hari Ini")
        df_trans = pd.DataFrame(st.session_state.transactions)
        st.dataframe(df_trans, use_container_width=True)
        
        if st.button("Reset Riwayat Transaksi"):
            st.session_state.transactions = []
            st.rerun()

# --- TAB 2: KALKULATOR TAKSIRAN (MARGIN 10%) ---
with tab2:
    st.subheader("Panduan Harga Beli & Simulator Taksiran")
    
    st.markdown("##### 🔍 Cek Harga Beli Cepat")
    sim_karat = st.selectbox("Karat Barang untuk Ditaksir", [f"{k} Karat ({k}K)" for k in range(24, 0, -1)], key="sim_k")
    sim_berat = st.number_input("Berat Barang (Gram) untuk Ditaksir", min_value=0.01, value=5.0, step=0.1, key="sim_b")
    
    sk_num = int(sim_karat.split(" ")[0])
    sk_pct = 0.99 if sk_num == 24 else (0.875 if sk_num == 22 else (0.75 if sk_num == 18 else (0.70 if sk_num == 17 else sk_num / 24.0)))
    
    s_pasaran_g = harga_24k * sk_pct
    s_beli_g = s_pasaran_g * (1 - 0.10)
    s_total_beli = s_beli_g * sim_berat
    s_profit = (s_pasaran_g * sim_berat) - s_total_beli
    
    st.info(f"""
    * **Kadar Persentase:** `{sk_pct*100:.1f}%`
    * **Harga Pasaran / Gram:** `Rp {s_pasaran_g:,.0f}`
    * **🎯 Harga Beli Maksimal / Gram (Margin 10%):** `Rp {s_beli_g:,.0f}`
    * **💰 Total Maksimal Bayar ke Pelanggan:** `Rp {s_total_beli:,.0f}`
    * **📈 Estimasi Profit Toko:** `Rp {s_profit:,.0f}`
    """)

# --- TAB 3: REKAP & KAS HARIAN ---
with tab3:
    st.subheader("Ringkasan Kas & Audit Harian")
    
    if st.session_state.transactions:
        df_rekap = pd.DataFrame(st.session_state.transactions)
        total_kas_keluar = df_rekap["Total Bayar"].sum()
        total_berat_masuk = df_rekap["Berat (g)"].sum()
        total_trx = len(df_rekap)
        total_profit_est = df_rekap["Estimasi Profit"].sum()
        
        col1, col2 = st.columns(2)
        with col1:
            st.metric("Total Kas Keluar", f"Rp {total_kas_keluar:,.0f}")
            st.metric("Total Transaksi", f"{total_trx} Transaksi")
        with col2:
            st.metric("Total Berat Masuk", f"{total_berat_masuk:.2f} Gram")
            st.metric("Estimasi Total Profit", f"Rp {total_profit_est:,.0f}")
    else:
        st.warning("Belum ada transaksi yang dicatat hari ini.")

