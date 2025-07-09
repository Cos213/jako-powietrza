# jakość-powietrza

## mój pomysl na projekt:
bot znający województwa podający jakość powietrza

#kod 

import discord
from discord.ext import commands, tasks
import aiohttp
import asyncio

TOKEN = "token"
CHANNEL_ID = 1390331392096866355

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)


miasto_wojewodztwo = {
    "Warszawa": "mazowieckie",
    "Kraków": "małopolskie",
    "Wrocław": "dolnośląskie",
    "Poznań": "wielkopolskie",
    "Gdańsk": "pomorskie",
    "Lublin": "lubelskie",
    "Katowice": "śląskie",
    "Rzeszów": "podkarpackie",
    "Olsztyn": "warmińsko-mazurskie",
    "Bydgoszcz": "kujawsko-pomorskie",
    "Zielona Góra": "lubuskie",
    "Białystok": "podlaskie",
    "Opole": "opolskie",
    "Kielce": "świętokrzyskie",
    "Łódź": "łódzkie",
    "Szczecin": "zachodniopomorskie",
}


wojewodztwa = list(set(miasto_wojewodztwo.values()))

@bot.event
async def on_ready():
    print(f"✅ Zalogowano jako {bot.user}")
    check_air_quality.start()

@bot.command()
async def powietrze(ctx, *, wojewodztwo: str = None):
    if not wojewodztwo:
        await ctx.send("ℹ️ Użycie: `!powietrze <nazwa_województwa>` (np. `!powietrze małopolskie`)")
        return

    wojewodztwo = wojewodztwo.lower()
    if wojewodztwo not in wojewodztwa:
        await ctx.send(f"❌ Nieznane województwo: `{wojewodztwo}`.\nDostępne: {', '.join(wojewodztwa)}")
        return

    await ctx.send(f"🔍 Szukam jakości powietrza dla **{wojewodztwo}**...")
    await sprawdz_powietrze(ctx.send, wojewodztwo)

@tasks.loop(hours=24)
async def check_air_quality():
    channel = bot.get_channel(CHANNEL_ID)
    if not channel:
        return

    for woj in wojewodztwa:
        await sprawdz_powietrze(channel.send, woj)
        await asyncio.sleep(2)  

async def sprawdz_powietrze(send_func, wojewodztwo):
    async with aiohttp.ClientSession() as session:
        try:
           
            url = "https://api.gios.gov.pl/pjp-api/v3/api-docs"
            async with session.get(url) as resp:
                if resp.status != 200:
                    await send_func(f"❌ Błąd pobierania listy stacji (status {resp.status})")
                    return
                stations = await resp.json()

            found = False
            for station in stations:
                city = station.get("city", {}).get("name")
                voivodeship = station.get("city", {}).get("commune", {}).get("voivodeshipName", "").lower()

                if voivodeship != wojewodztwo:
                    continue

                station_id = station["id"]

                
                index_url = f"https://api.gios.gov.pl/pjp-api/rest/aqindex/getIndex/{station_id}"
                async with session.get(index_url) as index_resp:
                    if index_resp.status != 200:
                        continue
                    index_data = await index_resp.json()
                    level = index_data.get("stIndexLevel", {}).get("indexLevelName", "brak danych")

                
                data_url = f"https://api.gios.gov.pl/pjp-api/rest/aqdata/getData/{station_id}"
                async with session.get(data_url) as data_resp:
                    if data_resp.status != 200:
                        continue
                    data_json = await data_resp.json()
                    values = data_json.get("values", [])

                    
                    pm10 = next((v["value"] for v in values if v["param"]["paramCode"] == "PM10" and v["value"] is not None), None)
                    pm25 = next((v["value"] for v in values if v["param"]["paramCode"] == "PM25" and v["value"] is not None), None)

                emoji = {
                    "Bardzo dobry": "🟢",
                    "Dobry": "🟡",
                    "Umiarkowany": "🟠",
                    "Dostateczny": "🔴",
                    "Zły": "⚫",
                    "Bardzo zły": "🧨",
                    "brak danych": "❔"
                }.get(level, "❔")

                msg = f"{emoji} **{station['stationName']}** ({city}): `{level}`"
                if pm10 is not None:
                    msg += f", PM10: {pm10} µg/m³"
                if pm25 is not None:
                    msg += f", PM2.5: {pm25} µg/m³"

                await send_func(msg)
                found = True
                await asyncio.sleep(1) 

            if not found:
                await send_func(f"😕 Nie znaleziono stacji w województwie **{wojewodztwo}** lub brak danych.")
        except Exception as e:
            await send_func(f"⚠️ Wystąpił błąd: {e}")

try:
    bot.run(TOKEN)
except Exception as e:
    print(f"❌ Błąd uruchomienia bota: {e}")
