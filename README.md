# Custom BlessCheck example

this is an example for a custom bless check

To make your own, you need 2 outputs: "True, '_' " and 'False, reason'  

create an extension and call it custom_bless. Leave everything untouched and create a file called cog.py in that folder. Then, create a bless_check() function like that ⬇️⬇️⬇️


as an example:  
```py
from bd_blessings.models import bless_settings, Blessings, Emojis, create_blessing

def bless_check(interaction: discord.Interaction):
    if True:
        await create_blessing(interaction.user.id)
        return True, "_"
    else:
        return False, "true wasnt true"
```

this is the only thing you need, but you can go all out and (for example add SKU's)

After that, just add it to the extra.toml and rebuild. 
Here's a full example, but you can also check cog.py aswell  
  
<details>
<summary>Full example:</summary>
  
```py
from bd_blessings.models import bless_settings, Blessings, Emojis, create_blessing
import discord


async def bless_check(interaction: discord.Interaction):

    max_blessings = 25

    last = await sync_to_async(
        lambda: Blessings.objects.filter(user=interaction.user.id).order_by("-id").first()
    )()

    if last is not None and last.claimed_time is not None:
        elapsed = timezone.now() - last.claimed_time
        if elapsed < timedelta(weeks=1):
            remaining = timedelta(weeks=1) - elapsed
            days = remaining.days
            hours = remaining.seconds // 3600
            return False, f"you can bless again in {days}d {hours}h"

    active_count = await sync_to_async(
        lambda: Blessings.objects.filter(active=True).count()
    )()
    if active_count >= max_blessings:
        return False, f"there are already {max_blessings} active blessings, try again later"

    view = await EmojiSelectView.create()
    await interaction.followup.send(view=view)

    await view.wait()
    if not view.confirmed:
        return False, "no emoji selected or confirmation timed out"

    return True, "_"

class EmojiSelectView(discord.ui.LayoutView):
    def __init__(self, emojis: list[Emojis]):
        super().__init__()
        self.emoji: int
        self.selected: discord.PartialEmoji = None
        self.emojis = emojis

        self.build()
    
    def build(self):
        emojis = self.emojis
        selected = self.selected
        select = discord.ui.Select(
            custom_id="bless_emoji_select",
            placeholder="Emoji",
            options=[
                discord.SelectOption(
                    label=emoji.name,
                    value=str(emoji.pk),
                    emoji=discord.PartialEmoji(name="_", id=emoji.emoji_id),
                )
                for emoji in emojis[:25]
            ],
        )
        select.callback = self.on_select
        self.container1 = discord.ui.Container(
            discord.ui.TextDisplay(content="## Select an emoji!"),
            discord.ui.TextDisplay(content="select the emoji that will appear alongside this blessing!"),
            discord.ui.Separator(visible=True, spacing=discord.SeparatorSpacing.large),
            discord.ui.ActionRow(select),
            discord.ui.Separator(visible=True, spacing=discord.SeparatorSpacing.small),
            discord.ui.TextDisplay(content=f"Selected: {self.selected if self.selected else "None"}"),
        )

        confirm_button = discord.ui.Button(style=discord.ButtonStyle.success, label="confirm",)
        confirm_button.callback = self.confirm_bless

        action_row1 = discord.ui.ActionRow(confirm_button)
        self.add_item(self.container1)
        self.add_item(action_row1)
        action_row1

    @classmethod
    async def create(cls):
        emojis = await sync_to_async(list)(Emojis.objects.all())
        return cls(emojis)

    async def on_select(self, interaction: discord.Interaction):
        emoji_num = int(interaction.data["values"][0])
        emoji = await sync_to_async(Emojis.objects.get)(pk=emoji_num)
        self.emoji = emoji
        self.selected = discord.PartialEmoji(name="_", id=self.emoji.emoji_id)
        self.clear_items()
        self.build()


        await interaction.response.edit_message(view=self)

        await interaction.response.defer(thinking=True, ephemeral=True)
        await interaction.followup.send("Woag, good pick")

    async def confirm_bless(self, interaction: discord.Interaction):
        if self.emoji:
            await create_blessing(interaction.user.id, self.emoji.emoji_id)
        else:
            await create_blessing(interaction.user.id)
        self.confirmed = True
        self.clear_items()
        self.add_item(discord.ui.TextDisplay(content=f"ok thanks"))
        await interaction.response.edit_message(view=self)
        
        self.stop()
```  
</details>  
  

.  
wow! you did it! Now, just add this into the extra.toml:  
```toml
[[ballsdex.packages]]
location = "/code/extra/CustomBless" # edit if its on github
path = "custom_bless"
enabled = true
editable = true # also edit if its on github
```