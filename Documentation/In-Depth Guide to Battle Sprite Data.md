Whether you want to make an existing class use a weapon type it didn't have access to before, or are replacing one of the existing classes with a different one entirely, it's important to be able to tell the game how to display this new class and weapon combination during battle. And the means to do that is through the Battle Sprite data table and the secondary data tables it references.

(Note that all ROM addresses given in this guide will be for an unheadered ROM. Also, please remember that any multibyte data in the ROM is stored with the lowest byte first, so you'll need to flip the byte order to do any kind of calculations on it.)

Battle Sprite Data Table

Unless expanded by repointing, the Battle Sprite data table contains 206 8 byte long entries starting at address $1788C9. In order, each entry contains one byte for the class number, one byte for the weapon type, one byte for the character number, one byte for the gender, one byte for an animation number, one byte for a main sprite number, one byte for a secondary sprite number, and one byte for a palette number.

The information in the first 4 bytes is used by the game to decide which entry to use, but the code that searches for the correct entry doesn't check the values in the order they're listed in - it starts by checking the class value, but then checks the gender second and the character number third, with the weapon type being checked last. Also, the generic values for gender ($FF) and character number ($00) being present in an entry will cause the check for matching gender and matching character number respectively to be skipped. So when changing the entries in the Battle Sprite data table, they need to stay grouped by class and then by gender, with character specific entries listed before generic entries.

The numbers in the last 4 bytes are used to locate information in other tables regarding data for how to animate the sprite, the main images used for the sprite, the secondary images used for the sprite, and the color palette(s) to use to color the images. The main sprite, secondary sprite, and palette numbers are all used to retrieve information from one other table, but the animation number is used for at least two tables.

I won't be going into details about the animation tables at this time because I'm still investigating them, but I will do a breakdown of the other tables.

Main Sprite Data Table

The Main Sprite data table, which starts at address $178039, consists of two sections, the first of which has single byte entries that are used to convert the main sprite number listed in the Battle Sprite data table entry to an entry index for the second section. Finding the correct entry in the first section from the main sprite number isn't quite as simple as you'd initially think, though - the main sprite number has one added to it before being used to get the relevant entry from the first section, so instead of main sprite number $00 corresponding to the first entry in the first section as you'd expect, it actually corresponds to the second entry.

The entries in the second section are 5 bytes long, with the first 3 bytes containing the address where the compressed image data is stored and the last 2 bytes containing $FFFF. (I did do a bit of looking into the code that gets reached if the last 2 bytes aren't $FFFF - it seems to read some data from the Battle Palette data table. Given that all the actual entries leave this value as $FFFF, however, it's probably incomplete/unused functionality the developers left in. I'd recommend not messing with it and just having this value be $FFFF in any new or edited entries for this table.)

To go from the address listed in the table to where the compressed image data is actually located in the ROM, you need to subtract $C00000 from it (and add $200 if your ROM is headered). (Except for the Barbarian, where you only subtract $800000.) Of course, this only tells you where the compressed image data starts, not where it ends, but the main and secondary battle sprite image data is mostly stored consecutively, with the image data for one sprite ending right before another main or secondary battle sprite's data begins.

An important note is that if the compressed data would extend past the boundary from the first half of a data bank to the second half of a data bank, the data doesn't continue past the end of the first half into the second half. Instead, the remainder of the data will be stored starting at the beginning of the next data bank. (This caused me some issues when I was trying to extract some of this compressed image data from the ROM until I realized that the data wasn't always in one continuous chunk.)

(Also, if you haven't encountered the term data bank before, SNES ROMs are divided into chunks that are $10000 bytes long called banks. So addresses between $000000 - $00FFFF would belong to one bank and the addresses between $010000 - $01FFFF would belong to another one, etc. The first half of data bank $00 would be from $000000 to $007FFF while the second half would be from $008000 to $00FFFF. And all of these adresses would be offset by $200 if you're working with a headered ROM.)

Secondary Sprite Data Table

The Secondary Sprite data table, which starts at address $17828B, is structured similarly to the Main Sprite data table, with an initial section that's used to convert the secondary sprite number listed in the Battle Sprite data table entry to an entry index for the second section. As with the Main Sprite data table, you need to add one to the secondary sprite number to find the correct entry.

The entries in the second section are 7 bytes long this time, starting with 2 bytes that I'm not sure of the purpose of, 3 bytes containing the address where the compressed image data is stored, and 2 bytes containing $FFFF again. The first 2 bytes are $8080 for all of the entries except the ballista sprites, which have $60A0 there instead. If you're changing or adding entries to this table, I'd recommend using the value used by other similar classes for the first 2 bytes.

You may be wondering why the image data for the battle sprites are split in two. For infantry classes, the main sprite contains the character, while the secondary sprite contains the character's shadow and any weapon trails that might be needed to show the character attacking with the relevant weapon. Mounted classes, meanwhile, have the mount in the main sprite, while the rider is in the secondary sprite along with the shadow and weapon trails that an infantry class would have. The ballista classes have the bow part of the ballista in the main sprite, while the base and operator are in the secondary sprite.

Battle Palette Data Table

The Battle Palette data table starts at address $1784B6 and, like the Main and Secondary Sprite data tables, is split into two sections with the entries in the first section acting as indices to entries in the second section.

The entries in the second section are 4 bytes long, 3 bytes for the address where the palette is stored and 1 byte for how many bytes of data to read from that address. The data stored at the listed address is not compressed, unlike the sprites, and takes 2 bytes to store 1 color. Palettes for infantry units read $1E bytes of data (so 15 colors) and palettes for mounted and ballista units read $3E bytes of data (so 31 colors). Why more colors for the mounted and ballista units? It's so they can use different colors for the secondary sprite image than are used in the main sprite image. Infantry units just have shadows and weapon trails in their secondary sprite images, which can use the main sprite palette and the weapon's palette respectively.

A note about the 16 colors used by mounted and ballista units - the first color listed is always used for the background of the sprite image and treated as transparent, so they're still limited to 15 actually usable colors for their secondary sprites.

Also, multiple palettes of the correct size can be stored at the address listed in an entry. If there are multiple palettes, they're used in the order of player -> enemy -> ally -> neutral, but not all entries have all of those palettes - most often there will just be player and enemy versions. Unfortunately, there isn't a good way to tell how many palettes are stored at a particular address other than checking where the next set of palettes start.

Custom Battle Palette Data Table

While not directly referenced by the Battle Sprite data table, the Custom Battle Palette data table that starts at address $1786AD is relevant because it is checked first to see if a character should use a different palette from the standard one referenced in the Battle Sprite table.

This table isn't divided into two sections like the Main Sprite, Secondary Sprite, and Palette data tables. Its entries are 4 bytes long, with 2 bytes for the character number, 1 byte for the class number, and 1 byte for a palette number. The last entry in the table isn't the full 4 bytes in length, just 2 bytes containing $FFFF to indicate the end of the table. The palette numbers listed in these entries are used to find entries in the Battle Palette data table, just like the palette numbers in the Battle Sprite data table.

Repointing These Tables

All of the references to these particular tables appear to be in a set of subroutines that are called when a battle starts, making it relatively simple to move one or more of these tables to a different location in the ROM and update the references to point to the new location. There are also places where the length of a table or a section of a table is used, which will also need to be updated if you expand a table.

If the location you move a table to is within the 4 MB of the original ROM, you'll need to add an offset of $C00000 to the address for it to point to the correct data. If you are working with a ROM that has been expanded to 8 MB (if you're working with a ROM that has been patched with the Project Naga patch, for example) and you moved a table to the expanded area of the ROM, you can just used the address without adding an offset.

The locations in the ROM that need to be updated and what needs to go there are listed in the tables below:

**Battle Sprite Data Table**

|Starting Address | Length in Bytes | Value|
|--- | --- | ---|
|$15A4E2 | 3 | Battle Sprite Data Table Starting Address|
|$15A4FF | 3 | Battle Sprite Data Table Starting Address + 3|
|$15A514 | 3 | Battle Sprite Data Table Starting Address + 2|
|$15A520 | 3 | Battle Sprite Data Table Starting Address + 2|
|$15A53D | 3 | Battle Sprite Data Table Starting Address + 1|
|$15A54F | 2 | Length of Battle Sprite Data Table in bytes|
|$15A556 | 3 | Battle Sprite Data Table Starting Address + 4|
|$15A55E | 3 | Battle Sprite Data Table Starting Address + 5|
|$15A566 | 3 | Battle Sprite Data Table Starting Address + 6|
|$15A56E | 3 | Battle Sprite Data Table Starting Address + 7|
|$15A579 | 3 | Battle Sprite Data Table Starting Address - 4|
|$15A581 | 3 | Battle Sprite Data Table Starting Address - 3|
|$15A589 | 3 | Battle Sprite Data Table Starting Address - 2|
|$15A591 | 3 | Battle Sprite Data Table Starting Address - 1|
|$15A59C | 3 | Battle Sprite Data Table Starting Address + 4|
|$15A5A4 | 3 | Battle Sprite Data Table Starting Address + 5|
|$15A5AC | 3 | Battle Sprite Data Table Starting Address + 6|
|$15A5B4 | 3 | Battle Sprite Data Table Starting Address + 7|

**Main Sprite Data Table**

|Starting Address | Length in Bytes | Value|
|--- | --- | ---|
|$1586E2 | 3 | Main Sprite Data Table Starting Address|
|$1586F0 | 2 | Length of Main Sprite Data Table Starting Address|
|$1586F5 | 3 | Main Sprite Data Table Starting Address + 1|
|$1586FB | 3 | Main Sprite Data Table Starting Address|
|$158727 | 3 | Main Sprite Data Table Starting Address + 3|
|$158738 | 3 | Main Sprite Data Table Starting Address + 3|

**Secondary Sprite Data Table**

|Starting Address | Length in Bytes | Value|
|--- | --- | ---|
|$15860B | 3 | Secondary Battle Sprite Data Table Starting Address|
|$15861A | 2 | Length of Secondary Battle Sprite Data Table in bytes|
|$158633 | 3 | Secondary Battle Sprite Data Table Starting Address + 3|
|$158639 | 3 | Secondary Battle Sprite Data Table Starting Address + 2|
|$15864F | 3 | Secondary Battle Sprite Data Table Starting Address + 5|
|$158660 | 3 | Secondary Battle Sprite Data Table Starting Address + 5|
|$1586AA | 3 | Secondary Battle Sprite Data Table Starting Address|
|$1586C4 | 3 | Secondary Battle Sprite Data Table Starting Address + 1|

**Battle Palette Data Table**

|Starting Address | Length in Bytes | Value|
|--- | --- | ---|
|$15866E | 3 | Battle Sprite Palette Data Table Starting Address + 1|
|$158674 | 3 | Battle Sprite Palette Data Table Starting Address|
|$158746 | 3 | Battle Sprite Palette Data Table Starting Address + 1|
|$15874C | 3 | Battle Sprite Palette Data Table Starting Address|
|$1587E9 | 3 | Battle Sprite Palette Data Table Starting Address|
|$1587F3 | 2 | Length of the Battle Sprite Data Table in bytes|
|$1587F7 | 3 | Battle Sprite Palette Data Table Starting Address|
|$1587FD | 3 | Battle Sprite Palette Data Table Starting Address + 1|
|$158803 | 3 | Battle Sprite Palette Data Table Starting Address + 3|

**Custom Battle Palette Data Table**

|Starting Address | Length in Bytes | Value|
|--- | --- | ---|
|$1587AD | 3 | Custom Battle Palette Data Table|
|$1587BA | 3 | Custom Battle Palette Data Table + 2|
|$1587CD | 3 | Custom Battle Palette Data Table + 3|
