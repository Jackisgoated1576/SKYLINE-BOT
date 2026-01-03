# SKYLINE-BOT
# ------------------- CONFIG -------------------

BOT_TOKEN =  MTQ1NjkyMzl2MDcyMDY0ODl1OA.GcokH5.BJVnv1yhPjT94w3rK3eUWfrL7cMcXeFv1ch5BQ # <-- Replace with your new token

ONLINE_ATC_CHANNEL_ID = 123456789012345678  # <-- Replace with your Online ATC channel ID

FLIGHT_PLAN_CHANNEL_ID = 987654321098765432  # <-- Replace with your Flight Plans channel ID

ATC_ROLE_NAME = "ATC"  # The name of the role your ATC staff will have

# ----------------------------------------------



intents = discord.Intents.default()

intents.members = True

intents.voice_states = True

intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)



# Track ATC claims and flight plan sessions

atc_claims = {}  # {"Tower": user_id, ...}

fp_sessions = {}  # {user_id: {"step":1, "data":{}}}

left_vc = {}

ATC_POSITIONS = ["Tower", "Ground", "Approach", "Departure", "Center"]



# ------------------- ATC CLAIM SYSTEM -------------------

@bot.event

async def on_voice_state_update(member, before, after):

    # Only ATC role users

    atc_role = discord.utils.get(member.guild.roles, name=ATC_ROLE_NAME)

    if not atc_role or atc_role not in member.roles:

        return



    # If user leaves VC, start 2-min timer to auto-release

    if before.channel is not None and after.channel is None:

        left_vc[member.id] = asyncio.create_task(auto_release(member))

    elif after.channel is not None and member.id in left_vc:

        left_vc[member.id].cancel()

        del left_vc[member.id]



async def auto_release(member):

    try:

        await asyncio.sleep(120)  # 2 minutes

        atc_role = discord.utils.get(member.guild.roles, name=ATC_ROLE_NAME)

        await member.remove_roles(atc_role)

        online_channel = bot.get_channel(ONLINE_ATC_CHANNEL_ID)

        released_positions = [pos for pos, uid in atc_claims.items() if uid == member.id]

        for pos in released_positions:

            del atc_claims[pos]

        await online_channel.send(f"⚠️ ATC position released: {member.mention} (inactive >2 min)")

        del left_vc[member.id]

    except asyncio.CancelledError:

        pass



@bot.command()

async def claim(ctx, position: str):

    position = position.title()

    online_channel = bot.get_channel(ONLINE_ATC_CHANNEL_ID)

    atc_role = discord.utils.get(ctx.guild.roles, name=ATC_ROLE_NAME)

    if not atc_role:

        await ctx.send(f"❌ ATC role '{ATC_ROLE_NAME}' does not exist.")

        return



    if position not in ATC_POSITIONS:

        await ctx.send(f"❌ Invalid position. Choose from: {', '.join(ATC_POSITIONS)}")

        return



    if position in atc_claims:

        await ctx.send(f"❌ {position} is already claimed by <@{atc_claims[position]}>")

        return



    await ctx.author.add_roles(atc_role)

    atc_claims[position] = ctx.author.id

    await online_channel.send(f"✅ ATC {position}: {ctx.author.mention}")



@bot.command()

async def release(ctx):

    online_channel = bot.get_channel(ONLINE_ATC_CHANNEL_ID)

    atc_role = discord.utils.get(ctx.guild.roles, name=ATC_ROLE_NAME)

    user_positions = [pos for pos, uid in atc_claims.items() if uid == ctx.author.id]



    if not user_positions:

        await ctx.send("❌ You are not claiming any ATC positions.")

        return



    await ctx.author.remove_roles(atc_role)

    for pos in user_positions:

        del atc_claims[pos]

        await online_channel.send(f"⚠️ ATC {pos} released by {ctx.author.mention}")



@bot.command()

async def online(ctx):

    online_channel = bot.get_channel(ONLINE_ATC_CHANNEL_ID)

    message = "📡 **Current ATC Online Positions:**\n"

    for pos in ATC_POSITIONS:

        if pos in atc_claims:

            user = ctx.guild.get_member(atc_claims[pos])

            message += f"{pos}: {user.mention}\n"

        else:

            message += f"{pos}: free\n"

    await online_channel.send(message)



# ------------------- FLIGHT PLAN SYSTEM -------------------

@bot.command()

async def fp(ctx):

    if ctx.author.id in fp_sessions:

        await ctx.send("❌ You already have an ongoing flight plan session.")

        return

    fp_sessions[ctx.author.id] = {"step":1, "data":{}}

    await ctx.send("✈️ Flight Plan Creation Started! What is your **Callsign**?")



@bot.event

async def on_message(message):

    await bot.process_commands(message)

    if message.author.bot:

        return



    user_id = message.author.id

    if user_id not in fp_sessions:

        return



    session = fp_sessions[user_id]

    step = session["step"]

    data = session["data"]

    content = message.content.strip()

    flight_channel =
