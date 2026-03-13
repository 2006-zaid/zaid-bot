# -*- coding: utf-8 -*-

import asyncio
import logging
import os
import re
import time
import tempfile
from collections import defaultdict
from datetime import date
from typing import Optional

import aiohttp
import aiofiles
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, BotCommand
from telegram.constants import ParseMode, ChatAction
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

TOKEN = "8797865004:AAG_BWHOew0wMNhWbHsN0_c2k_1Jq4iEHNY"

OWNER_ID = 6493670464

RATE_LIMIT_MAX = 5
RATE_LIMIT_WINDOW = 60
DUPLICATE_COOLDOWN = 600

DOWNLOAD_DIR = tempfile.gettempdir()

TIKTOK_RE = re.compile(r"(https?://)?(www\.|vm\.|vt\.)?tiktok\.com/\S+", re.I)
YOUTUBE_RE = re.compile(r"(https?://)?(www\.)?(youtube\.com|youtu\.be)/\S+", re.I)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

rate_limit_store = defaultdict(lambda: {"count": 0, "window_start": 0})
duplicate_store = defaultdict(dict)

stats = {
    "total_users": set(),
    "total_downloads": 0,
    "daily": defaultdict(int),
}


def detect_platform(url):
    if TIKTOK_RE.search(url):
        return "tiktok"
    if YOUTUBE_RE.search(url):
        return "youtube"
    return None


def is_rate_limited(user_id):
    now = time.time()
    data = rate_limit_store[user_id]

    if now - data["window_start"] > RATE_LIMIT_WINDOW:
        data["count"] = 0
        data["window_start"] = now

    data["count"] += 1
    return data["count"] > RATE_LIMIT_MAX


def is_duplicate(user_id, url):
    now = time.time()
    links = duplicate_store[user_id]

    duplicate_store[user_id] = {u: t for u, t in links.items() if now - t < DUPLICATE_COOLDOWN}

    if url in duplicate_store[user_id]:
        return True

    duplicate_store[user_id][url] = now
    return False


def build_buttons(url):
    return InlineKeyboardMarkup([
        [
            InlineKeyboardButton("🎬 Video", callback_data=f"video|{url}"),
            InlineKeyboardButton("🎵 Audio", callback_data=f"audio|{url}")
        ]
    ])


SHARE_URL = (
    "https://t.me/share/url"
    "?url=https://t.me/zaid_cf_bot"
    "&text=🎬 Download TikTok and YouTube videos instantly.%0A%0A"
    "🎵 Video or Audio — your choice!%0A"
    "⚡ Fast, free, and easy to use."
)

def menu_buttons(user_id):
    buttons = [
        [InlineKeyboardButton("❓ Help", callback_data="help")],
        [InlineKeyboardButton("🔗 Share Bot", url=SHARE_URL)],
        [InlineKeyboardButton("👨‍💻 Developer", url="https://t.me/TlIT25")]
    ]
    if user_id == OWNER_ID:
        buttons.insert(1, [InlineKeyboardButton("📊 Statistics", callback_data="stats")])
    return InlineKeyboardMarkup(buttons)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    stats["total_users"].add(user.id)

    text = (
        f"👋 Welcome {user.first_name}\n\n"
        "Send a link from:\n"
        "🎵 TikTok\n"
        "▶️ YouTube\n\n"
        "Choose Video or Audio."
    )

    await update.message.reply_text(text, reply_markup=menu_buttons(user.id))


async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Send a TikTok or YouTube link.\n"
        "Then choose Video or Audio."
    )


async def link_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    url = update.message.text.strip()

    if is_rate_limited(user.id):
        await update.message.reply_text("Too many requests. Wait a minute.")
        return

    platform = detect_platform(url)

    if not platform:
        await update.message.reply_text("Only TikTok and YouTube links are supported.")
        return

    if is_duplicate(user.id, url):
        await update.message.reply_text("You already sent this link recently.")
        return

    label = {"tiktok": "TikTok", "youtube": "YouTube"}[platform]

    await update.message.reply_text(
        f"{label} link detected.\nChoose download type:",
        reply_markup=build_buttons(url)
    )


async def download(url, audio=False):
    import yt_dlp

    opts = {
        "outtmpl": os.path.join(DOWNLOAD_DIR, "%(id)s.%(ext)s"),
        "quiet": True,
        "noplaylist": True,
    }

    if audio:
        opts["format"] = "bestaudio/best"
        opts["postprocessors"] = [{
            "key": "FFmpegExtractAudio",
            "preferredcodec": "mp3"
        }]
    else:
        opts["format"] = "best"

    loop = asyncio.get_event_loop()

    def run():
        with yt_dlp.YoutubeDL(opts) as ydl:
            info = ydl.extract_info(url, download=True)
            return ydl.prepare_filename(info)

    return await loop.run_in_executor(None, run)


async def buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    data = query.data
    user_id = query.from_user.id

    if data == "help":
        await query.edit_message_text("Send TikTok or YouTube link.")
        return

    if data == "stats":
        if user_id != OWNER_ID:
            await query.answer("⛔ This section is for the owner only.", show_alert=True)
            return
        today = str(date.today())
        await query.edit_message_text(
            f"📊 *Bot Statistics*\n\n"
            f"👥 Total Users: {len(stats['total_users'])}\n"
            f"📥 Total Downloads: {stats['total_downloads']}\n"
            f"📅 Today's Downloads: {stats['daily'][today]}",
            parse_mode=ParseMode.MARKDOWN
        )
        return

    if "|" not in data:
        return

    action, url = data.split("|", 1)

    msg = await query.edit_message_text("Downloading...")

    try:
        file = await download(url, audio=(action == "audio"))

        async with aiofiles.open(file, "rb") as f:
            file_data = await f.read()

        import io

        if action == "audio":
            await query.message.chat.send_audio(io.BytesIO(file_data))
        else:
            await query.message.chat.send_video(io.BytesIO(file_data))

        os.remove(file)

        stats["total_downloads"] += 1
        stats["daily"][str(date.today())] += 1

        await msg.delete()

        await query.message.reply_text(
            "✅ Done! Enjoy your download 🎉\n\n"
            "💡 *Share this bot with your friends and let them enjoy it too!*\n"
            "👇 Press the button below to share ⬇️",
            parse_mode=ParseMode.MARKDOWN,
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("🔗 Share Bot with Friends", url=SHARE_URL)]
            ])
        )

    except Exception as e:
        logger.error(e)
        await msg.edit_text("Download failed.")


def main():

    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, link_handler))
    app.add_handler(CallbackQueryHandler(buttons))

    print("Bot running...")

    app.run_polling()


if __name__ == "__main__":
    main()
