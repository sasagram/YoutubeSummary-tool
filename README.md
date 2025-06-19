import streamlit as st
import os
import requests
from datetime import datetime, timedelta, timezone
import time
import google.generativeai as genai
from notion_client import Client
from openai import OpenAI
from typing import List, Dict, Optional
import yt_dlp
import json
import re
import toml

# -----------------------------------------------------------------
# 画面のセットアップ & 定数定義
# -----------------------------------------------------------------
st.set_page_config(
    page_title="YouNo - YouTube to Notion Automator",
    page_icon="🎉",
    layout="wide"
)

SECRETS_PATH = os.path.join(".streamlit", "secrets.toml")
PROMPTS_PATH = "prompts.json"

LOGO_SVG = """
<div style="text-align: center; margin-bottom: 20px;">
    <svg width="280" height="100" viewBox="0 0 280 100" xmlns="http://www.w3.org/2000/svg">
        <defs><style>@import url('https://fonts.googleapis.com/css2?family=Inter:wght@700&display=swap');</style></defs>
        <rect x="0" y="10" width="130" height="80" rx="16" fill="#FF0000"/>
        <text x="65" y="65" font-family="'Inter', sans-serif" font-size="36" font-weight="700" text-anchor="middle" fill="white" letter-spacing="-0.02em">You</text>
        <rect x="142" y="10" width="130" height="80" rx="16" fill="white" stroke="#e2e8f0" stroke-width="3"/>
        <text x="207" y="65" font-family="'Inter', sans-serif" font-size="36" font-weight="700" text-anchor="middle" fill="#2d3748" letter-spacing="-0.02em">No</text>
    </svg>
</div>
"""

PRESET_PROMPTS = {
    "トピック別・詳細分析（推奨）": """
あなたはプロのビジネスアナリストです。与えられたYouTube動画のテキストを分析し、その内容をトピックごとに深く掘り下げ、構造化して要約するのがあなたの仕事です。
【指示】
1. まず、以下の【テキスト】から、動画の主要なトピックを複数、特定してください。
2. 特定した各トピックを「####（H4見出し）」として、それぞれ独立したセクションを作成してください。
3. 各トピックのセクション内では、必ず以下の4つの【必須項目】について、太字（**）で囲み、その下に箇条書き（-）で具体的に記述してください。
    - **【内容】**: そのトピックが「何であるか」を具体的に説明します。
    - **【メリット】**: そのトピックがもたらす利点や、ポジティブな側面を説明します。
    - **【デメリット・注意点】**: そのトピックの欠点や、考慮すべきリスク、注意点を客観的に説明します。
    - **【業務効率化のポイント】**: そのトピックの情報を、実際の業務や個人の生産性向上にどう活かせるか、具体的なアクションを提案します。
4. もし特定の項目に該当する内容がなければ、箇条書きで「特になし」と記述してください。
5. チャンネル紹介、挨拶、定型的な宣伝文句など、本質的でない内容は完全に要約から除外してください。
【テキスト】
{text_to_summarize}
""",
    "タイムテーブル基準要約": """
あなたは優秀なアシスタントです。与えられた【チャプターリスト】を骨格として、【全文文字起こし】の内容を、各チャプターに沿って要約してください。
【指示】
1. 【チャプターリスト】の各項目を、そのまま「####（H4見出し）」として出力してください。
2. 各チャプターの見出しの下に、対応する【全文文字起こし】の部分から要点を抽出し、箇条書き（-）で簡潔にまとめてください。
3. 文字起こしの内容がチャプターと無関係な場合や、宣伝のみの場合は、そのチャプターの要約は省略するか、「特になし」と記述してください。
【チャプターリスト】
{chapter_list}

【全文文字起こし】
{text_to_summarize}
"""
}

# --- バックエンド関数群 ---
def add_video_to_notion(video_info: Dict, notion_client, db_id: str, log_area):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            log_area.info(f"  [Notion] ページ追加中... (試行 {attempt + 1}/{max_retries})")
            children_blocks = [{"object": "block", "type": "embed", "embed": {"url": video_info["url"]}}, {"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "AIによる要約"}}]}},]
            summary = video_info.get("summary", "")
            if "#### " in summary:
                topics = filter(None, summary.strip().split("\n#### "))
                for topic_text in topics:
                    parts = topic_text.split('\n', 1)
                    topic_title = parts[0].strip()
                    topic_content_str = parts[1].strip() if len(parts) > 1 else ""
                    
                    nested_toggles = []
                    # **【...】** のパターンで分割
                    sub_sections = re.split(r'(\*\*.+?\*\*)', topic_content_str)
                    sub_sections = [s for s in sub_sections if s.strip()] # 空の要素を削除
                    
                    for i in range(0, len(sub_sections), 2):
                        if i + 1 < len(sub_sections):
                            sub_heading = sub_sections[i].replace("*", "").replace("【", "").replace("】", "").strip()
                            sub_content = sub_sections[i+1].strip()
                            if sub_heading and sub_content:
                                nested_toggles.append({"object": "block", "type": "toggle", "toggle": {"rich_text": [{"type": "text", "text": {"content": sub_heading}}], "children": [{"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": sub_content}}]}}]}})
                    
                    children_blocks.append({"object": "block", "type": "toggle", "toggle": {"rich_text": [{"type": "text", "text": {"content": topic_title}}], "children": nested_toggles if nested_toggles else None}})
            elif summary:
                children_blocks.append({"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": summary}}]}})
            
            children_blocks.extend([{"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "文字起こし"}}]}}])
            if video_info.get("transcript"):
                for i in range(0, len(video_info.get("transcript", "")), 2000): children_blocks.append({"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": video_info["transcript"][i:i + 2000]}}]}})
            else:
                children_blocks.append({"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": "この動画の文字起こしは利用できませんでした。"}}]}})
            
            notion_client.pages.create(parent={"database_id": db_id}, properties={"タイトル": {"title": [{"text": {"content": video_info["title"]}}]},"チャンネル": {"rich_text": [{"text": {"content": video_info["channel_title"]}}]},"投稿日": {"date": {"start": video_info["published_at"]}},"動画URL": {"url": video_info["url"]},"ステータス": {"select": {"name": "未読"}},}, children=children_blocks)
            log_area.success(f"  ★[Notion] 動画「{video_info['title']}」をNotionに追加しました。"); return True
        except Exception as e:
            log_area.warning(f"  [Notion] 追加中にエラーが発生 (試行 {attempt + 1}): {e}");
            if attempt < max_retries - 1: time.sleep((attempt + 1) * 5)
            else: log_area.error(f"  [Notion] 最大再試行回数に達しました。")
    return False

def get_channel_videos(channel_id: str, start_date, end_date, api_key: str, log_area) -> List[Dict]:
    log_area.info(f"▶ [YouTube] チャンネルID: {channel_id} ({start_date.strftime('%Y-%m-%d')} ~ {end_date.strftime('%Y-%m-%d')})"); videos = []
    try:
        published_after = datetime.combine(start_date, datetime.min.time(), tzinfo=timezone.utc).isoformat().replace('+00:00', 'Z')
        published_before = datetime.combine(end_date + timedelta(days=1), datetime.min.time(), tzinfo=timezone.utc).isoformat().replace('+00:00', 'Z')
        url = f"https://www.googleapis.com/youtube/v3/search?key={api_key}&channelId={channel_id}&part=snippet,id&order=date&maxResults=50&type=video&publishedAfter={published_after}&publishedBefore={published_before}"
        response = requests.get(url); response.raise_for_status(); data = response.json()
        for item in data.get("items", []):
            if item.get("id", {}).get("kind") == "youtube#video" and "videoId" in item.get("id", {}):
                videos.append({"id": item["id"]["videoId"], "title": item["snippet"]["title"], "description": item["snippet"]["description"], "url": f"https://www.youtube.com/watch?v={item['id']['videoId']}", "published_at": datetime.fromisoformat(item["snippet"]["publishedAt"].replace('Z', '+00:00')).strftime('%Y-%m-%dT%H:%M:%SZ'),"channel_title": item["snippet"]["channelTitle"], "original_published_at": datetime.fromisoformat(item["snippet"]["publishedAt"].replace('Z', '+00:00'))})
        log_area.info(f"   └ {len(videos)}件の動画データを取得しました。"); return videos
    except Exception as e: log_area.error(f"   └ APIリクエストエラー: {e}"); return []

def get_transcript_with_ytdlp(video_url: str, log_area) -> Optional[str]:
    log_area.info(f"[yt-dlp] 文字起こし取得試行... (URL: {video_url})"); output_folder = "summary"; ydl_opts = {'writesubtitles': True, 'writeautomaticsub': True, 'subtitleslangs': ['ja', 'en'], 'skip_download': True, 'outtmpl': f'{output_folder}/%(id)s.subtitle', 'quiet': True, 'no_warnings': True,};
    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(video_url, download=False); video_id = info.get('id'); requested_subtitles = info.get('requested_subtitles')
            if not requested_subtitles: log_area.warning("[yt-dlp] この動画には利用可能な文字起こしが見つかりませんでした。"); return None
            ydl.download([video_url]); subtitle_lang = list(requested_subtitles.keys())[0]; ext = requested_subtitles[subtitle_lang]['ext']; subtitle_filename = f"{output_folder}/{video_id}.subtitle.{subtitle_lang}.{ext}"
            if os.path.exists(subtitle_filename):
                with open(subtitle_filename, 'r', encoding='utf-8') as f:
                    lines = f.readlines()
                    transcript_text = " ".join([line.strip() for line in lines if not "-->" in line and not line.strip().isdigit() and not "<c>" in line and line.strip()])
                os.remove(subtitle_filename); log_area.success(f"★[yt-dlp] 文字起こし取得成功！ ({subtitle_lang})"); return transcript_text
            else: log_area.warning(f"[yt-dlp] 文字起こしファイルが見つかりませんでした。({subtitle_filename})"); return None
    except Exception as e: log_area.error(f"[yt-dlp] 文字起こし取得中にエラーが発生しました: {e}"); return None

def summarize_with_ai(ai_model, text_to_summarize, api_key, prompt_template, chapter_list, log_area):
    # チャプターがある場合とない場合でプロンプトを決定
    if chapter_list:
        final_prompt = PRESET_PROMPTS["タイムテーブル基準要約"].format(chapter_list=chapter_list, text_to_summarize=text_to_summarize)
    else:
        final_prompt = prompt_template.format(text_to_summarize=text_to_summarize)

    # 選択されたAIモデルに応じて処理を分岐
    if ai_model == "Gemini":
        try:
            genai.configure(api_key=api_key)
            log_area.info("[Gemini] 構造化要約を生成中..."); model = genai.GenerativeModel('gemini-1.5-flash-latest'); response = model.generate_content(final_prompt); summary = response.text
            log_area.success("[Gemini] 要約生成完了。"); return summary.strip() if summary else "要約の生成に失敗しました。"
        except Exception as e: log_area.error(f"[Gemini] 要約中にエラーが発生しました: {e}"); return f"Gemini APIエラー: {e}"
    
    elif ai_model == "OpenAI":
        try:
            client = OpenAI(api_key=api_key)
            log_area.info("[OpenAI] GPTで要約を生成中...")
            response = client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": final_prompt}])
            summary = response.choices[0].message.content
            log_area.success("[OpenAI] 要約生成完了。"); return summary.strip() if summary else "要約の生成に失敗しました。"
        except Exception as e: log_area.error(f"[OpenAI] 要約中にエラーが発生しました: {e}"); return f"OpenAI APIエラーにより要約できませんでした: {e}"
    
    return None

def extract_chapters_from_description(description: str) -> Optional[str]:
    pattern = re.compile(r"(\d{1,2}:\d{2}(?::\d{2})?)\s*(.*)")
    chapters = pattern.findall(description)
    if not chapters: return None
    return "\n".join([f"- {timestamp} {title.strip()}" for timestamp, title in chapters])

def test_notion_connection(token, db_id):
    if not token or not db_id: return "トークンとデータベースIDを入力してください。"
    try: notion = Client(auth=token); notion.databases.retrieve(database_id=db_id); return "✅ 接続に成功しました！"
    except Exception as e: return f"❌ 接続に失敗しました: {e}"

def test_youtube_connection(api_key):
    if not api_key: return "APIキーを入力してください。"
    try:
        url = f"https://www.googleapis.com/youtube/v3/search?key={api_key}&part=snippet&maxResults=1&q=test"; response = requests.get(url)
        if response.status_code == 200: return "✅ 接続に成功しました！"
        else: return f"❌ 接続に失敗しました: {response.json().get('error', {}).get('message', '不明なエラー')}"
    except Exception as e: return f"❌ 接続に失敗しました: {e}"

def test_gemini_connection(api_key):
    if not api_key: return "APIキーを入力してください。"
    try:
        genai.configure(api_key=api_key)
        if [m for m in genai.list_models() if 'generateContent' in m.supported_generation_methods]: return "✅ 接続に成功しました！"
        else: return "❌ 接続に成功しましたが、利用可能なモデルがありません。"
    except Exception as e: return f"❌ 接続に失敗しました: {e}"

def test_openai_connection(api_key):
    if not api_key: return "APIキーを入力してください。"
    try: client = OpenAI(api_key=api_key); client.models.list(); return "✅ 接続に成功しました！"
    except Exception as e: return f"❌ 接続に失敗しました: {e}"

def load_secrets():
    if os.path.exists(SECRETS_PATH):
        with open(SECRETS_PATH, "r") as f: return toml.load(f)
    return {}

def save_secrets(secrets_data):
    os.makedirs(".streamlit", exist_ok=True)
    with open(SECRETS_PATH, "w") as f: toml.dump(secrets_data, f)

def load_prompts():
    if os.path.exists(PROMPTS_PATH):
        with open(PROMPTS_PATH, "r", encoding='utf-8') as f: return json.load(f)
    return {}

def save_prompts(prompts_data):
    with open(PROMPTS_PATH, "w", encoding='utf-8') as f: json.dump(prompts_data, f, ensure_ascii=False, indent=2)

def start_process(api_keys, channel_ids, start_date, end_date, ai_model, prompt_template):
    with st.container():
        log_area = st.status("処理待機中...", state="running", expanded=True)
        try:
            log_area.info("設定ファイルを読み込み中...");
            try:
                with open('config.json', 'r', encoding='utf-8') as f: config = json.load(f)
                rules = config.get("filter_rules", {}); include_keywords, exclude_keywords, interval_seconds = rules.get("title_include_keywords", []), rules.get("title_exclude_keywords", []), config.get("execution_interval_seconds", 5)
            except FileNotFoundError: log_area.warning("config.jsonが見つかりません。フィルタリングはスキップされます。"); include_keywords, exclude_keywords, interval_seconds = [], [], 5
            
            notion_client = Client(auth=api_keys["notion_token"]); all_videos = []
            for channel_id in channel_ids:
                videos = get_channel_videos(channel_id, start_date, end_date, api_keys["youtube"], log_area); all_videos.extend(videos)
            
            if not all_videos: log_area.success("期間内に投稿された動画はありませんでした。"); st.balloons(); return
            
            log_area.info(f"取得した全 {len(all_videos)} 件の動画をフィルタリング..."); videos_to_process = []
            for video in all_videos:
                title_lower = video["title"].lower()
                if exclude_keywords and any(key.lower() in title_lower for key in exclude_keywords): log_area.info(f"[フィルタ] 除外: '{video['title']}'"); continue
                if include_keywords and not any(key.lower() in title_lower for key in include_keywords): log_area.info(f"[フィルタ] 除外: '{video['title']}'"); continue
                if is_video_processed(video["url"], notion_client, api_keys["notion_db_id"]): log_area.info(f"[フィルタ] スキップ: '{video['title']}'"); continue
                videos_to_process.append(video)
            
            if not videos_to_process: log_area.success("フィルタリングの結果、処理対象の動画はありませんでした。"); st.balloons(); return
            
            log_area.info(f"全 {len(videos_to_process)} 件の動画処理を開始..."); videos_to_process.sort(key=lambda x: x["original_published_at"])
            
            for video in videos_to_process:
                log_area.info(f"処理対象: {video['title']}")
                video["transcript"] = get_transcript_with_ytdlp(video["url"], log_area)
                text_to_summarize = video["transcript"] or f"タイトル: {video['title']}\n説明:\n{video['description']}"
                chapter_list = extract_chapters_from_description(video["description"])
                if chapter_list: log_area.info(f"  [Info] 動画のチャプターを検出しました。チャプター基準で要約します。")
                
                api_key = api_keys.get('gemini') if ai_model == "Gemini" else api_keys.get('openai')
                video["summary"] = summarize_with_ai(ai_model, text_to_summarize, api_key, prompt_template, chapter_list, log_area)
                
                add_video_to_notion(video, notion_client, api_keys["notion_db_id"], log_area)
                log_area.info(f"待機中 ({interval_seconds}秒)..."); time.sleep(interval_seconds)
            
            log_area.success(f"完了！合計 {len(videos_to_process)} 件の新しい動画を処理しました。"); st.balloons()
        except Exception as e: log_area.error(f"予期せぬエラーが発生しました: {e}")

# --- session_state の初期化 ---
if 'channel_ids' not in st.session_state:
    try:
        with open('config.json', 'r', encoding='utf-8') as f: config = json.load(f)
        st.session_state.channel_ids = config.get("youtube_channel_ids", [""])
    except FileNotFoundError: st.session_state.channel_ids = [""]
if 'custom_prompts' not in st.session_state:
    st.session_state.custom_prompts = load_prompts()
if 'prompt_template_content' not in st.session_state:
    st.session_state.prompt_template_content = PRESET_PROMPTS["トピック別・詳細分析（推奨）"]

# --- UI定義 ---
with st.sidebar:
    st.header("💡 使い方ガイド")
    st.markdown("（このエリアは、プロジェクトの最後に完成させます）")
    st.divider()
    st.header("📚 各種ヘルプ")
    with st.expander("YouTubeチャンネルIDの探し方", expanded=False):
        st.markdown("1. チャンネルのトップページを開き、URLの`.../channel/UCxxxx`の`UC...`部分をコピーします。\n2. `...@handle`形式の場合は、ページのソースから`channelId`で検索してください。")
    with st.expander("Notion APIキーの取得方法", expanded=False):
        st.markdown("1. [Notionインテグレーション](https://www.notion.so/my-integrations)で作成します。\n2. DBとコネクト連携も忘れずに。")
    with st.expander("YouTube Data APIキーの取得方法", expanded=False):
        st.markdown("1. [Google Cloud Console](https://console.cloud.google.com/apis/library/youtube.googleapis.com)で有効化し、作成します。")
    with st.expander("Gemini APIキーの取得方法", expanded=False):
        st.markdown("1. [Google AI Studio](https://aistudio.google.com/app/apikey)で作成します。")
    with st.expander("OpenAI APIキーの取得方法", expanded=False):
        st.markdown("1. [OpenAI Platform](https://platform.openai.com/api-keys)にアクセスし、新しいキーを作成します。")

st.markdown(LOGO_SVG, unsafe_allow_html=True)

secrets = load_secrets()
api_secrets = secrets.get("api_keys", {})

with st.expander("▼ STEP1: APIキーと使用モデルの設定", expanded=False):
    st.subheader("① 使用するAIモデル")
    ai_model = st.selectbox("要約に使用するAIを選択", ["Gemini", "OpenAI"])
    
    st.subheader("② APIキー")
    c1, c2 = st.columns(2)
    with c1:
        st.write("**Notion & YouTube**")
        notion_token = st.text_input("Notionトークン", value=api_secrets.get("notion_token", ""), type="password")
        notion_db_id = st.text_input("NotionデータベースID", value=api_secrets.get("notion_db_id", ""), type="password")
        youtube_api_key = st.text_input("YouTube APIキー", value=api_secrets.get("youtube_api_key", ""), type="password")
    with c2:
        if ai_model == "Gemini":
            st.write("**Google AI (Gemini)**")
            gemini_api_key = st.text_input("Gemini APIキー", value=api_secrets.get("gemini_api_key", ""), type="password", key="gemini_key")
            openai_api_key = api_secrets.get("openai_api_key", "") # 他方のキーは保持
            if st.button("Gemini 接続テスト"): st.info(test_gemini_connection(gemini_api_key))
        elif ai_model == "OpenAI":
            st.write("**OpenAI (ChatGPT)**")
            openai_api_key = st.text_input("OpenAI APIキー", value=api_secrets.get("openai_api_key", ""), type="password", key="openai_key")
            gemini_api_key = api_secrets.get("gemini_api_key", "") # 他方のキーは保持
            if st.button("OpenAI 接続テスト"): st.info(test_openai_connection(openai_api_key))

    if st.button("全てのAPIキーをファイルに保存"):
        save_secrets({"api_keys": {"notion_token": notion_token, "notion_db_id": notion_db_id, "youtube_api_key": youtube_api_key, "gemini_api_key": gemini_api_key, "openai_api_key": openai_api_key}})
        st.success("APIキーを `.streamlit/secrets.toml` に保存しました！")

with st.expander("▼ STEP2: ワークフロー設定", expanded=True):
    col1_ws, col2_ws = st.columns(2)
    with col1_ws:
        st.subheader("① 監視対象チャンネル")
        for i, channel_id in enumerate(st.session_state.channel_ids):
            c1_inner, c2_inner = st.columns([0.9, 0.1]); st.session_state.channel_ids[i] = c1_inner.text_input(f"ID #{i+1}", value=channel_id, label_visibility="collapsed", key=f"channel_{i}")
            if c2_inner.button("➖", key=f"del_{i}"): st.session_state.channel_ids.pop(i); st.rerun()
        if st.button("＋ チャンネルを追加"): st.session_state.channel_ids.append(""); st.rerun()
    with col2_ws:
        st.subheader("② 検索期間")
        today = datetime.now().date()
        start_date = st.date_input("検索開始日", value=today - timedelta(days=6))
        end_date = st.date_input("検索終了日", value=today)

with st.expander("▼ STEP3: AIへの命令（プロンプト）設定", expanded=False):
    st.subheader("① プロンプトテンプレート選択")
    prompt_options = ["---- プリセット ----"] + list(PRESET_PROMPTS.keys()) + ["---- マイテンプレート ----"] + list(st.session_state.custom_prompts.keys())
    selected_prompt_name = st.selectbox("テンプレートを選択", options=prompt_options, help="ここからAIへの命令の雛形を選べます。")
    
    if selected_prompt_name in PRESET_PROMPTS:
        st.session_state.prompt_template_content = PRESET_PROMPTS[selected_prompt_name]
    elif selected_prompt_name in st.session_state.custom_prompts:
        st.session_state.prompt_template_content = st.session_state.custom_prompts[selected_prompt_name]

    prompt_template = st.text_area("命令文（プレビュー＆編集エリア）", value=st.session_state.prompt_template_content, height=300)
    
    st.subheader("② マイテンプレート管理")
    with st.form("template_form"):
        template_name = st.text_input("テンプレート名")
        submitted = st.form_submit_button("この内容でマイテンプレートを新規保存・上書き保存")
        if submitted:
            if template_name and prompt_template:
                st.session_state.custom_prompts[template_name] = prompt_template
                save_prompts(st.session_state.custom_prompts)
                st.success(f"マイテンプレート「{template_name}」を保存しました！")
                time.sleep(1); st.rerun()
            else:
                st.warning("テンプレート名と内容を両方入力してください。")
    
    del_template_name = st.selectbox("削除するマイテンプレートを選択", options=[""] + list(st.session_state.custom_prompts.keys()))
    if del_template_name:
        if st.button(f"選択中の「{del_template_name}」を削除", type="secondary"):
            del st.session_state.custom_prompts[del_template_name]
            save_prompts(st.session_state.custom_prompts)
            st.success(f"「{del_template_name}」を削除しました！")
            time.sleep(1); st.rerun()

st.divider()
run_button = st.button("処理を開始する", type="primary", use_container_width=True)

if run_button:
    st.header("処理ログ")
    api_keys = {"notion_token": notion_token, "notion_db_id": notion_db_id, "youtube": youtube_api_key, "gemini": gemini_api_key, "openai": openai_api_key}
    
    if not all([notion_token, notion_db_id, youtube_api_key, (gemini_api_key if ai_model == 'Gemini' else openai_api_key)]): st.error("エラー: 必要なAPIキーをSTEP1で入力・保存してください。")
    elif not any(cid.strip() for cid in st.session_state.channel_ids): st.error("エラー: 監視対象のチャンネルIDをSTEP2で1つ以上入力してください。")
    else:
        valid_channels = [cid for cid in st.session_state.channel_ids if cid.strip()]
        if not valid_channels: st.error("エラー: 有効なチャンネルIDが入力されていません。")
        else:
            start_process(
                api_keys=api_keys, 
                channel_ids=valid_channels, 
                start_date=start_date, 
                end_date=end_date,
                ai_model=ai_model, 
                prompt_template=prompt_template
            )  
