
import asyncio
import discord
from discord.ext import commands

# =====================================================================
# 1. BOT CONFIGURATION
# =====================================================================

# PASTE YOUR TOKEN INSIDE THE QUOTES BELOW:
BOT_TOKEN = "MTU2NjgwNzcwMjA5OTg1MzMyMQ.GlCr60.vYOcp2lmSbkNXPE6N6S4egXh3y3Hu0M-uZhdR8"

# Your Staff Role ID (1567026563369664590)
STAFF_ROLE_IDS = [1567026563369664590]

# Your Ticket Category ID (1567026565483859991)
TICKET_CATEGORY_ID = 1567026565483859991

# Configuration for your 4 ticket buttons
TICKET_TYPES = {
    "support": {
        "title": "General Support",
        "prefix": "support",
        "color": discord.Color.blurple(),
        "welcome": "Please describe what you need help with, and a staff member will assist you shortly.",
        "roles": STAFF_ROLE_IDS,
    },
    "report": {
        "title": "Report a Member",
        "prefix": "report",
        "color": discord.Color.red(),
        "welcome": "Please provide the **username/ID** of the member you are reporting, along with screenshots or evidence.",
        "roles": STAFF_ROLE_IDS,
    },
    "appeal": {
        "title": "Ban or Mute Appeal",
        "prefix": "appeal",
        "color": discord.Color.gold(),
        "welcome": "Please share why you were muted/banned and why you believe the punishment should be lifted.",
        "roles": STAFF_ROLE_IDS,
    },
    "partnership": {
        "title": "Partnership Request",
        "prefix": "partner",
        "color": discord.Color.green(),
        "welcome": "Please share your server/project details, member count, and what kind of partnership you are looking for.",
        "roles": STAFF_ROLE_IDS,
    },
}


# =====================================================================
# 2. INSIDE-TICKET CONTROLS (Claim & Close Buttons)
# =====================================================================

class TicketControlView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self._claim_lock = asyncio.Lock()

    def is_staff(self, member: discord.Member) -> bool:
        if member.guild_permissions.administrator:
            return True
        return any(role.id in STAFF_ROLE_IDS for role in member.roles)

    async def safe_reply(self, interaction: discord.Interaction, text: str):
        """Replies safely even if Android background lag caused the 3s interaction token to expire."""
        try:
            if not interaction.response.is_done():
                await interaction.response.send_message(text, ephemeral=True)
            else:
                await interaction.followup.send(text, ephemeral=True)
        except Exception:
            pass

    @discord.ui.button(
        label="Claim Ticket",
        style=discord.ButtonStyle.success,
        emoji="🙋‍♂️",
        custom_id="ticket_control:claim",
    )
    async def claim_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        if not self.is_staff(interaction.user):
            await self.safe_reply(interaction, "❌ Only staff members can claim tickets.")
            return

        # Lock prevents two staff members from claiming at the exact same millisecond
        async with self._claim_lock:
            # Fetch fresh message state so simultaneous clicks or expired interactions never fail
            try:
                msg = await interaction.channel.fetch_message(interaction.message.id)
            except Exception:
                msg = interaction.message

            if not msg or not msg.embeds:
                await self.safe_reply(interaction, "❌ Could not find the ticket embed.")
                return

            embed = msg.embeds[0]
            claim_field_value = embed.fields[1].value if len(embed.fields) > 1 else ""

            if button.disabled or "Unclaimed" not in claim_field_value:
                await self.safe_reply(
                    interaction,
                    "⚠️ This ticket has already been claimed by another staff member!",
                )
                return

            button.disabled = True
            button.label = f"Claimed by {interaction.user.display_name}"
            button.style = discord.ButtonStyle.secondary

            embed.set_field_at(
                index=1,
                name="Claimed By",
                value=interaction.user.mention,
                inline=True,
            )

            try:
                if not interaction.response.is_done():
                    await interaction.response.edit_message(embed=embed, view=self)
                else:
                    await msg.edit(embed=embed, view=self)
            except Exception:
                await msg.edit(embed=embed, view=self)

            await interaction.channel.send(
                f"✅ {interaction.user.mention} has claimed this ticket and will be assisting you!"
            )

    @discord.ui.button(
        label="Close Ticket",
        style=discord.ButtonStyle.danger,
        emoji="🔒",
        custom_id="ticket_control:close",
    )
    async def close_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        try:
            await interaction.response.send_message(
                f"🔒 Ticket closed by {interaction.user.mention}. Deleting channel in **5 seconds**..."
            )
        except Exception:
            await interaction.channel.send(
                f"🔒 Ticket closed by {interaction.user.mention}. Deleting channel in **5 seconds**..."
            )
        await asyncio.sleep(5)
        await interaction.channel.delete(reason=f"Ticket closed by {interaction.user}")


# =====================================================================
# 3. MAIN TICKET PANEL (4 Category Buttons)
# =====================================================================

class TicketPanelView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    async def open_ticket(self, interaction: discord.Interaction, type_key: str):
        # Try to defer within 3 seconds; if Android paused Pydroid and the token expired (10062),
        # do NOT crash — continue creating the ticket channel anyway!
        interaction_valid = True
        try:
            await interaction.response.defer(ephemeral=True)
        except Exception:
            interaction_valid = False

        guild = interaction.guild
        user = interaction.user
        config = TICKET_TYPES[type_key]

        for channel in guild.text_channels:
            if channel.topic and f"UserID:{user.id}" in channel.topic and f"Type:{type_key}" in channel.topic:
                if interaction_valid:
                    try:
                        await interaction.followup.send(
                            f"You already have an open **{config['title']}** ticket: {channel.mention}",
                            ephemeral=True,
                        )
                    except Exception:
                        pass
                return

        overwrites = {
            guild.default_role: discord.PermissionOverwrite(view_channel=False),
            user: discord.PermissionOverwrite(
                view_channel=True,
                send_messages=True,
                read_message_history=True,
                attach_files=True,
                embed_links=True,
            ),
            guild.me: discord.PermissionOverwrite(
                view_channel=True,
                send_messages=True,
                manage_channels=True,
                read_message_history=True,
                embed_links=True,
            ),
        }

        role_mentions = []
        for role_id in config["roles"]:
            role = guild.get_role(role_id)
            if role:
                overwrites[role] = discord.PermissionOverwrite(
                    view_channel=True,
                    send_messages=True,
                    read_message_history=True,
                    attach_files=True,
                    embed_links=True,
                )
                role_mentions.append(role.mention)

        category = guild.get_channel(TICKET_CATEGORY_ID)
        if not isinstance(category, discord.CategoryChannel):
            category = None

        channel_name = f"{config['prefix']}-{user.name.lower()}"

        # Retry helper to survive Android background connection drops ([Errno 103])
        ticket_channel = None
        for attempt in range(3):
            try:
                ticket_channel = await guild.create_text_channel(
                    name=channel_name,
                    category=category,
                    overwrites=overwrites,
                    topic=f"Ticket | Type:{type_key} | UserID:{user.id}",
                )
                break
            except OSError:
                await asyncio.sleep(1)

        if not ticket_channel:
            return

        embed = discord.Embed(
            title=f"🎫 {config['title']}",
            description=f"Welcome {user.mention}!\n\n{config['welcome']}",
            color=config["color"],
        )
        embed.add_field(name="Opened By", value=user.mention, inline=True)
        embed.add_field(name="Claimed By", value="Unclaimed", inline=True)
        embed.set_footer(text=f"User ID: {user.id}")

        pings_str = " ".join(role_mentions)

        # Retry sending the welcome embed up to 3 times if Android resets the socket
        for attempt in range(3):
            try:
                await ticket_channel.send(
                    content=f"{user.mention} {pings_str}".strip(),
                    embed=embed,
                    view=TicketControlView(),
                    allowed_mentions=discord.AllowedMentions(users=True, roles=True),
                )
                break
            except OSError:
                await asyncio.sleep(1)

        if interaction_valid:
            try:
                await interaction.followup.send(
                    f"✅ Your **{config['title']}** ticket has been created: {ticket_channel.mention}",
                    ephemeral=True,
                )
            except Exception:
                pass

    @discord.ui.button(
        label="General Support",
        style=discord.ButtonStyle.primary,
        emoji="💬",
        custom_id="ticket_panel:support",
        row=0,
    )
    async def support_btn(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.open_ticket(interaction, "support")

    @discord.ui.button(
        label="Report a Member",
        style=discord.ButtonStyle.danger,
        emoji="🚨",
        custom_id="ticket_panel:report",
        row=0,
    )
    async def report_btn(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.open_ticket(interaction, "report")

    @discord.ui.button(
        label="Ban or Mute Appeal",
        style=discord.ButtonStyle.secondary,
        emoji="⚖️",
        custom_id="ticket_panel:appeal",
        row=1,
    )
    async def appeal_btn(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.open_ticket(interaction, "appeal")

    @discord.ui.button(
        label="Partnership Request",
        style=discord.ButtonStyle.success,
        emoji="🤝",
        custom_id="ticket_panel:partnership",
        row=1,
    )
    async def partnership_btn(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.open_ticket(interaction, "partnership")


# =====================================================================
# 4. BOT SETUP & COMMANDS (!panel and /panel)
# =====================================================================

class TicketBot(commands.Bot):
    def __init__(self):
        intents = discord.Intents.default()
        intents.message_content = True
        super().__init__(command_prefix="!", intents=intents)

    async def setup_hook(self):
        self.add_view(TicketPanelView())
        self.add_view(TicketControlView())
        await self.tree.sync()


bot = TicketBot()


@bot.event
async def on_ready():
    print(f"✅ SUCCESS! Bot is ONLINE as: {bot.user}")


def create_panel_embed() -> discord.Embed:
    embed = discord.Embed(
        title="📬 Server Support & Tickets",
        description=(
            "Select one of the buttons below to open a private ticket with our staff team:\n\n"
            "💬 **General Support** — Ask questions or get help with the server.\n"
            "🚨 **Report a Member** — Report rule breakers with proof.\n"
            "⚖️ **Ban or Mute Appeal** — Appeal a punishment on your account.\n"
            "🤝 **Partnership Request** — Apply to partner with our server."
        ),
        color=discord.Color.blurple(),
    )
    embed.set_footer(text="Please do not open troll or duplicate tickets.")
    return embed


@bot.command(name="panel")
async def send_panel(ctx: commands.Context):
    await ctx.send(embed=create_panel_embed(), view=TicketPanelView())
    try:
        await ctx.message.delete()
    except discord.Forbidden:
        pass


@bot.tree.command(name="panel", description="Post the 4-button ticket panel in this channel")
async def slash_panel(interaction: discord.Interaction):
    await interaction.channel.send(embed=create_panel_embed(), view=TicketPanelView())
    await interaction.response.send_message("✅ Ticket panel posted!", ephemeral=True)


clean_token = BOT_TOKEN.strip()
if not clean_token or clean_token == "PASTE_YOUR_BOT_TOKEN_HERE":
    print("❌ ERROR: You forgot to paste your Discord Bot Token on Line 10!")
else:
    bot.run(clean_token)
    
