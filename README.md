# chatbot-test-hacktiv8
Testing chatbot with AI and delivered as Hacktiv8 Final Project

Streamlit berjalan: NgrokTunnel: "https://agency-wistful-survivor.ngrok-free.dev" -> "http://localhost:8501"

Reference:
* Hacktiv8 Tutorial
* Source repo: adiptamartulandi/chatbot-streamlit-demo

<hr>

<h2>STEP 1 — Install Streamlit dan pyngrok</h2>

```python
!pip install -q streamlit pyngrok google-genai
```

<h2>STEP 2 - Add NGROK Token into Codelab</h2>
Add your own NGROK token for tunneling into Google Codelab. Add it to your Codelab secret.

```python
from pyngrok import ngrok
from google.colab import userdata
ngrok.set_auth_token(userdata.get('NGROK_TOKEN'))
print("ngrok token berhasil dikonfigurasi!")
```

<h2>STEP 3 - Running Streamlit and Tunneling NGROK</h2>

```python
import subprocess
import time

def run_streamlit(filename, port=8501):
    # Kill SEMUA proses streamlit, bukan hanya yang kita spawn
    subprocess.run(["pkill", "-f", "streamlit"], capture_output=True)

    # Force-free port kalau masih ada yang nempel
    subprocess.run(["fuser", "-k", f"{port}/tcp"], capture_output=True)

    # Tutup semua tunnel ngrok
    ngrok.kill()

    # Tunggu port benar-benar bebas
    time.sleep(3)

    proc = subprocess.Popen(
        [
            "streamlit", "run", filename,
            "--server.headless=true",
            "--server.port", str(port),
            "--server.enableCORS=false",
        ],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL
    )

    time.sleep(3)

    public_url = ngrok.connect(port)
    print(f"Streamlit berjalan: {public_url}")

    return proc
```

<h2>STEP 4 - Making python file for its basic setting of AI Chatbot</h2>

This includes with slider parameter for its Temperature, topP and topK that can be set accordingly as user wants.

```python
%%writefile streamlit_chat_app.py
# Import library yang dibutuhkan
import streamlit as st          # framework web app
from google import genai         # SDK Gemini dari Google

# ── 1. Konfigurasi Halaman ───────────────────────────────────────────────────
# st.title() dan st.caption() menampilkan judul dan keterangan di bagian atas
st.title("Wiki Builder Chatbot")
st.caption("Chatbot sederhana untuk asistensi optimalisasi pembuatan konten pada Wiki. Powered by Gemini Flash.")

# ── 2. Sidebar: Pengaturan App ───────────────────────────────────────────────
# Semua widget di dalam blok 'with st.sidebar:' akan muncul di panel samping
with st.sidebar:
    st.subheader("Pengaturan")

    # Kotak input untuk API key
    # type="password" menyembunyikan teks yang diketik (muncul sebagai titik-titik)
    google_api_key = st.text_input("Google AI API Key", type="password")

    # Tombol untuk mereset percakapan
    # Parameter 'help' menampilkan tooltip saat kursor diarahkan ke tombol
    reset_button = st.button("Reset Percakapan", help="Hapus semua pesan dan mulai dari awal")

    temperature = st.sidebar.slider("Temperature", 0.0, 1.0, 0.7)
    top_p = st.sidebar.slider("Top P", 0.0, 1.0, 0.95)
    top_k = st.sidebar.slider("Top K", 0, 50, 40)
# ── 3. Validasi API Key ──────────────────────────────────────────────────────
# Kalau user belum memasukkan API key, tampilkan pesan dan hentikan eksekusi
if not google_api_key:
    st.info("Masukkan Google AI API Key di sidebar untuk mulai chat.", icon="🗝️")
    # st.stop() menghentikan eksekusi skrip di titik ini
    # Kode setelah st.stop() tidak akan dijalankan
    st.stop()

# ── 4. Inisialisasi Gemini Client ────────────────────────────────────────────
# Bagian ini hanya membuat client baru kalau:
# - Client belum pernah dibuat (pertama kali app dijalankan), ATAU
# - User mengganti API key di sidebar
#
# Kenapa perlu dicek seperti ini?
# Karena setiap interaksi user menyebabkan seluruh skrip dijalankan ulang.
# Tanpa pengecekan ini, kita akan membuat client baru setiap kali user ketik pesan
# — yang artinya konteks percakapan akan hilang terus.
#
# getattr(obj, 'attr', default) = cara aman mengakses atribut yang mungkin belum ada
if ("genai_client" not in st.session_state) or (
    getattr(st.session_state, "_last_key", None) != google_api_key
):
    try:
        # Buat client Gemini baru dengan API key dari sidebar
        st.session_state.genai_client = genai.Client(api_key=google_api_key)

        # Simpan key yang dipakai — untuk deteksi perubahan key nanti
        st.session_state._last_key = google_api_key

        # Kalau key berganti, hapus session chat lama
        # .pop() menghapus key dari dict dengan aman (tidak error kalau key tidak ada)
        st.session_state.pop("chat", None)
        st.session_state.pop("messages", None)

    except Exception as e:
        st.error(f"API Key tidak valid: {e}")
        st.stop()

# ── 5. Inisialisasi Chat Session & Riwayat Pesan ────────────────────────────
# Inisialisasi chat session Gemini kalau belum ada
if "chat" not in st.session_state:
    # Buat chat session baru dengan model gemini-2.5-flash
    # Session ini menyimpan konteks percakapan di sisi Gemini
    st.session_state.chat = st.session_state.genai_client.chats.create(
        model="gemini-2.5-flash",
        config={"system_instruction": "Kamu adalah asisten yang hanya menjawab kebutuhan tentang manajerial konten platform wiki. Sebagai asisten platform wiki, Anda sangat penting untuk menguasai Module dan Template pada MediaWiki.",
                "temperature":temperature,
                "top_p":top_p,
                "top_k":top_k
                }
    )
    
# Inisialisasi list riwayat pesan kalau belum ada
# List ini menyimpan semua pesan untuk ditampilkan kembali saat skrip dijalankan ulang
if "messages" not in st.session_state:
    st.session_state.messages = []

# ── 6. Tombol Reset ──────────────────────────────────────────────────────────
# Kalau tombol reset diklik, hapus chat session dan riwayat pesan
if reset_button:
    st.session_state.pop("chat", None)
    st.session_state.pop("messages", None)
    # st.rerun() memaksa Streamlit me-refresh halaman dari awal
    # Ini akan menjalankan ulang seluruh skrip, dan karena session sudah dihapus,
    # chat baru akan dibuat di langkah 5
    st.rerun()

# ── 7. Tampilkan Riwayat Percakapan ─────────────────────────────────────────
# Loop ini menampilkan semua pesan yang sudah ada di session_state
# Setiap kali skrip dijalankan ulang, pesan-pesan ini ditampilkan kembali
for msg in st.session_state.messages:
    # st.chat_message() membuat bubble chat dengan role yang sesuai
    # role "user" = bubble di kanan, role "assistant" = bubble di kiri
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# ── 8. Input & Respons ───────────────────────────────────────────────────────
# st.chat_input() membuat kotak input di bagian bawah halaman
# Nilai yang diketik user tersimpan di variabel 'prompt'
# Variabel ini bernilai None kalau user belum mengirim pesan
prompt = st.chat_input("Ketik pesanmu di sini...")

# Hanya jalankan bagian ini kalau user mengirim pesan
if prompt:
    # Langkah 1: Tambah pesan user ke riwayat
    st.session_state.messages.append({"role": "user", "content": prompt})

    # Langkah 2: Tampilkan bubble pesan user
    with st.chat_message("user"):
        st.markdown(prompt)

    # Langkah 3: Kirim ke Gemini dan tampilkan respons
    try:
        # Kirim pesan ke Gemini melalui chat session yang sudah ada
        # chat.send_message() secara otomatis menyertakan riwayat percakapan sebelumnya
        response = st.session_state.chat.send_message(prompt)

        # Ambil teks dari respons
        # hasattr() memeriksa apakah objek punya atribut tertentu
        # Ini mencegah error kalau format respons tidak terduga
        if hasattr(response, "text"):
            answer = response.text
        else:
            answer = str(response)

    except Exception as e:
        # Kalau ada error (misal: rate limit, koneksi putus), tampilkan pesan error
        answer = f"Terjadi error: {e}"

    # Langkah 4: Tampilkan bubble respons assistant
    with st.chat_message("assistant"):
        st.markdown(answer)

    # Langkah 5: Simpan respons ke riwayat
    st.session_state.messages.append({"role": "assistant", "content": answer})

```
<h2>STEP 5 - Running Chatbot </h2>

```python
proc = run_streamlit("streamlit_chat_app.py")
```

<h2>Exit - Stopping AI Chatbot</h2>

```python
# Hentikan Streamlit
try:
    proc.terminate()
    print("Streamlit dihentikan.")
except:
    print("Tidak ada proses yang berjalan.")

# Tutup semua tunnel ngrok
ngrok.kill()
print("Tunnel ngrok ditutup.")
```
