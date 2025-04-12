import discord
from discord.ext import commands
import os
from flask import Flask
from threading import Thread

# ===== Mini-Webserver für UptimeRobot =====
app = Flask('')

@app.route('/')
def home():
    return "Bot läuft!"

def run():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    Thread(target=run).start()

# ===== Umgebungsvariablen laden =====
TOKEN = os.getenv("TOKEN")
GUILD_ID = os.getenv("GUILD_ID")

if not TOKEN or not GUILD_ID:
    print("❌ TOKEN oder GUILD_ID fehlt in den Secrets! Bitte setze sie in Replit (Schlüssel-Symbol).")
    exit()

GUILD_ID = int(GUILD_ID)
CATEGORY_ID = 1360619368404357377  # Kategorie-ID für Tickets
SUPPORT_ROLE_ID = 1360280160993149300  # Support-Rollen-ID

# ===== Discord-Bot-Setup =====
intents = discord.Intents.default()
intents.messages = True
intents.guilds = True
intents.dm_messages = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)

# ===== Events =====
@bot.event
async def on_ready():
    print(f"✅ Bot ist online als {bot.user}")

@bot.event
async def on_message(message):
    if message.author == bot.user:
        return

    # Wenn DM vom User kommt
    if isinstance(message.channel, discord.DMChannel):
        guild = bot.get_guild(GUILD_ID)
        category = guild.get_channel(CATEGORY_ID)

        if category is None or not isinstance(category, discord.CategoryChannel):
            print("❌ Kategorie nicht gefunden oder ungültig.")
            return

        # Nach bestehendem Ticket suchen
        existing = None
        for channel in category.text_channels:
            if channel.topic == str(message.author.id):
                existing = channel
                break

        # Falls kein Ticket existiert → Neues erstellen
        if not existing:
            support_role = guild.get_role(SUPPORT_ROLE_ID)

            if support_role is None:
                print("❌ Support-Rolle nicht gefunden!")
                return

            overwrites = {
                guild.default_role: discord.PermissionOverwrite(read_messages=False),
                support_role: discord.PermissionOverwrite(read_messages=True, send_messages=True),
                guild.me: discord.PermissionOverwrite(read_messages=True, send_messages=True),
            }

            existing = await category.create_text_channel(
                f"ticket-{message.author.name}",
                topic=str(message.author.id),
                overwrites=overwrites
            )

            embed_intro = discord.Embed(
                title="<:assistent:1355601689947672636> Willkommen im Kundenservice",
                description=(
                    "Guten Tag,\n\n"
                    "Vielen Dank für das Kontaktieren unseres Kundenservices. "
                    "Hier beim ADAC sind wir immer für Sie da, egal um welches Anliegen es sich handelt.\n\n"
                    "In der Zwischenzeit bitten wir Sie um etwas Geduld, bis sich ein Mitarbeiter bei Ihnen meldet. "
                    "Bitte beschreiben Sie zunächst Ihr Anliegen."
                ),
                color=discord.Color.yellow()
            )
            await message.channel.send(embed=embed_intro)

            embed_ticket = discord.Embed(
                description=f"📩 Neues Ticket von **{message.author}** (ID: {message.author.id})",
                color=discord.Color.yellow()
            )
            await existing.send(embed=embed_ticket)

        # Nachricht ins Ticket leiten
        embed_user_msg = discord.Embed(
            description=f"**{message.author}**: {message.content}",
            color=discord.Color.yellow()
        )
        await existing.send(embed=embed_user_msg)

        embed_feedback = discord.Embed(
            description="<:check:1356648481862848715> Deine Nachricht wurde weitergeleitet. Bitte hab etwas Geduld, bis sich ein Mitarbeiter bei dir meldet.",
            color=discord.Color.yellow()
        )
        await message.channel.send(embed=embed_feedback)

    await bot.process_commands(message)

# ===== Commands =====

@bot.command()
async def antwort(ctx, *, nachricht):
    if ctx.channel.category_id != CATEGORY_ID:
        return
    user_id = ctx.channel.topic
    if user_id:
        user = await bot.fetch_user(int(user_id))
        try:
            embed_reply = discord.Embed(
                title="<:kommunikation:1355601694876106953>** Antwort vom Kundenservice**",
                description=nachricht,
                color=discord.Color.yellow()
            )
            await user.send(embed=embed_reply)

            confirm = discord.Embed(
                description="✅ Nachricht gesendet.",
                color=discord.Color.yellow()
            )
            await ctx.send(embed=confirm)
        except:
            error = discord.Embed(
                description="❌ Fehler beim Senden der Nachricht. Bitte Entwickler kontaktieren.",
                color=discord.Color.red()
            )
            await ctx.send(embed=error)

@bot.command()
async def close(ctx):
    if ctx.channel.category_id != CATEGORY_ID:
        return
    user_id = ctx.channel.topic
    if user_id:
        user = await bot.fetch_user(int(user_id))
        try:
            embed_close = discord.Embed(
                title="<:assistent:1355601689947672636> Ticket geschlossen",
                description=(
                    "Dein Kundenservice-Ticket wurde nun geschlossen. "
                    "Vielen Dank, dass du uns kontaktiert hast.\n\n"
                    "Bei weiteren Anliegen kannst du dich jederzeit wieder melden. "
                    "Wir wünschen dir weiterhin eine angenehme Fahrt."
                ),
                color=discord.Color.yellow()
            )
            await user.send(embed=embed_close)
        except:
            warning = discord.Embed(
                description="⚠️ Konnte den Nutzer nicht benachrichtigen.",
                color=discord.Color.orange()
            )
            await ctx.send(embed=warning)

    confirm = discord.Embed(
        description="🗑️ Dieses Ticket wird nun geschlossen.",
        color=discord.Color.yellow()
    )
    await ctx.send(embed=confirm)
    await ctx.channel.delete()

# ===== Start =====
keep_alive()
bot.run(TOKEN)
