Minecraft Expand
=============
A modification to Minecraft that generates user-created structures alongside terrain.

---
##Table of Contents
1. Project Introduction
2. Storing Structures
3. Modifying Existing Structures to work with Expand
4. Inserting Structures into Worlds
5. Developing a Structure Library
6. Development and Timeline
7. Conclusion

##1. Project Introduction
This concept is inspired by Reddit user [u/Hyta](http://www.reddit.com/user/Hyta)'s post found [here](http://www.reddit.com/r/minecraftsuggestions/comments/z2viq/minecraft_game_changer/).

Minecraft is incredibly extensive, and randomly generated structures that currently exist in game (temples, strongholds, villages, etc.) are fun to explore. This mod will aim to expand those default structures by allowing players to create their own structures and have them be randomly generated in their world. An example Hyta uses is a user-created bridge that will randomly generate across a ravine.

The next thing Hyta discusses in his post is a "structure gallery." As a part of this mod, I will be writing an online repository for uploading, managing, and distributing structure schematics and bundles. This is a core aspect of the mod, as it will allow other users to view what you've created and vice-versa.

##2. Storing Structures
Rather than re-invent the wheel, this mod will use what has become a Minecraft standard, the [schematic file format](http://minecraft.gamepedia.com/Schematic_file_format). `.schematic` files were created for tools such as MCEdit and are widely used, with thousands of currently existing schematics. The filetype is build on the NBT (Named Binary Tag) format, and therefore additional attributes can be added to the schematics without affecting its functionality in exisiting tools.

The modified schematic file format is the same as the standard, except with an added top-level attribute, `environment`. In pseudocode, the `environment` attribute is as follows:

```javascript
environment : {
    anchors: {
        NW: {
            bottom:      [blockId|predefined type ('ground','desert','rocky','water',etc.)],
            top:         [...],
            groundLayer: y_offset_from_structure_bottom | -1 for no ground
        },
        NE: { ... },
        SW: { ... },
        SE: { ... }
    },
    biomes: [<biome_number>|<biome_grouping>,...],
    heights: [(min,max),...]
}
```
I'll go a little more in depth on each attribute. 

####Anchors
The `anchor` attribute is based on the concept that each corner of the model acts as an "anchor" to the world that it is originally from. This anchor can then be used as a reference for which terrains the structure can placed. There are eight core anchors making up the eight vertices of a schematic's bounding rectangular prism, the bottom and top corners of the northwest, northeast, southwest, and southeast edges of the schematic. Each corner contains a list of valid blocks that will allow for the structure to generate. For example, if you wanted your structure to generate only on smooth stone, the corner would be a one-element list `bottom/top: [1]`. These lists can contain any valid blockId. Alternatively, the user can also use a predefined "type." For example, if the user wants the structure to generate on a desert-like terrain, they would just add `'desert'` to their list. These predefined types are just syntactical sugar for a list a associated blockIds. Furthering our current example, `'desert'` might represent the list `[12,24,43:1]`. This will likely lead to blocks being double counted. For now, reducing the blockIds to a unique list will be done at runtime, but the future holds opportunities to offer a compression tool of some sort.

Lastly, each anchor edge contains a `groundLayer` attribute. This is specifically included for structures built on hills. For objects built with no ground in mind (floating platforms, etc.), this value can be set to -1. Otherwise, it is the 0-indexed offset from the level below the bottom layer (0 meaning the structure sits naturally on top of the ground). Any value greater than or equal to the height of the schematic will be interpreted as an underground structure.

####Biomes
A very simple way to place a schematic is based on [biomes](http://minecraft.gamepedia.com/Biome). The `biomes` attribute will be a list of all biomes that the structure can show up in. The following values will be supported:
* Biome ID [int] - the numerical value, as found on the Minecraft Wiki, linked above.
* Biome Type [string] - the group classification, as found on the Minecraft Wiki (i.e. 'snowy'). There is one slight variation. Instead of accepting `'neutral'`, Expand divides this into two biome types, `'water'` and `'hills'`.
* 'all' [string] - Not recommended, as this will require the structure to be checked for every biome.

####Heights
The `heights` attribute is a list of valid heights where the structure can be found. The attribute is filled in the form of a list of tuples. Each tuple represents a `(min,max)` pair. The min is inclusive, while the max is exclusive. The attribute can contain multiple pairs, enabling greater control of the placement process. The min and max values will correlate to the valid starting layer of the schematic. That is, if a pair is `(0,5)`, the bottom of the schematic must fall within layers 0-5 of the world.

##3. Modifying Existing Structures to work with Expand
The design of Expand is to allow users to take already-generated content and edit it to use with Expand. In fact, for the first release, the only way to make Expand-compatible schematics will be to use the online conversion tool. Future releases might look to integrate directly into tools such as MCEdit through plugins. The process is fairly straightforward:

1. The user uploads a schematic to the Expand Schematic Converter
2. The converter reads in the file and validates it. It then displays a rough render of the schematic in the browser.
3. The converter prompts the user to set anchor blocks for each corner. The user will be able to choose from a combo box of blockIds and predefined block groups, as mentioned before.
4. The converter prompts the user to set the ground level at each corner edge.
5. The converter prompts the user to input valid height pairs.

From here, the user will be given an option to download their freshly-edited schematic or upload to the structure library.

##4. Inserting Structures into Worlds
This will be one of the most complex aspects of the project. A requirement is that world generation is not greatly affected, despite the fact that we are adding a lot to the terrain generation step. 

The mod will add two key settings to the user's Minecraft menu. The first is a "density" of structures. If this is set to low, users will see sporadic structures in their world. On the opposite end, if this is set to max, structures will generate nearly as often as possible. I intend to make these presets [low,medium,high,max], which will require some tuning. Currently I am envisioning these being set to occurences per chunk grouping. For example, low would mean structures have to be at least 20 chunks from the nearest structure, while high allows for a structure in every chunk. 

The second key setting is toggling which structures are able to generate. The user should be able to toggle both on the bundle and individual structure level.

Before the world starts generating, the mod will need to compact any inefficiencies that may have slipped through in schematic conversion (i.e. double counting anchor block IDs, etc.). At this step, syntactical sugar, such as predefined block groupings and biome groupings are reduced to a simple list of ids. This will not affect the schematic file and will occur only during runtime, as to maintain the user-friendly aspect. This way the program has a standard base to run from. Each included schematic is then added to a list of active schematics to check.

Every time a new chunk is loaded, several things must be done. This code needs to run immediately after the chunk has been loaded by Minecraft, as it requires the chunk block data to place the schematics. This is pseudocode for the process:
```no-highlight
if structure density is already fulfilled:
    move to next chunk
endIf
for each schematic:
    if the schematic's biome list contains the current chunk's biome:
        for each valid layer (found in the heights array):
            if all anchor data matches: 
                place structure
                store the position it was placed for density checking purposes
            endIf
        endFor
    endIf
endFor
```

Hopefully, this process will be able to be optimized enough that we do not encounter performance issues. Additionally, density and number of schematics will affect performance.

##5. Developing a Structure Library
As I mentioned before, the structure library is a core aspect of the mod. However, this is a secondary goal. The first step will be to ship with a default list of structures for alpha/beta testing. For the later part of beta testing, we will allow users to use the library system, so we can test that as well before going live. 

The library will consist of users creations, all converted into Expand-compatible *schematics*. Each schematic will be downloadable. Additionally, the library will support groupings schematics, called *bundles*.

###Schematics
Each schematic will use the standard schematic file format plus the additional Expand environment data. Additionally, the web interface will associate more detailed information on the schematic, including, but not limited to: 
* Name*
* Author
* Description*
* Date Created
* Last Updated
* User-uploaded image*
* Web-generated render
* Tags (i.e. bridge,house)*
* Downloads (direct + bundle)
* Rating
* Comments
* Flags

#####Creation
All schematics will first need to go through the converter. From there, the user can choose to download their modified schematic or continue the upload process. If the user already has a Expand-compatible schematic, they can immediately jump to the next step in the process.

Once the schematic file is uploaded, the user is presented with a simple form with fields for the above information with an asterisk(*) next to it. The other information will be auto-populated.

#####Editing/Deleting
Users will be able to edit and delete schematics. Editing will force us to support version-specific rating at some point, but that is for later down the line. Schematics won't be able to be truly deleted after a day grace period. After that day, rather than deleting, it will disassociate the structure from your account on the user side. However, the connection will remain so we can detect and eliminate malicious users. 

#####Downloading
Users will be able to browse schematics by attributes. To start, browsing will support a tag filtering system. This way users can easily find schematics relevant to their interests.


###Bundles
A bundle is a curated collection of schematics, usually along some sort of theme. Minecraft Expand will ship with two bundles, a _default_ bundle containing an "editor's choice" selection of schematics, and an empty _assorted_ bundle. The assorted bundle is a place where users can throw various converted schematics (singletons) they download. Bundles have a lot of data associated with them, similar to schematics. This includes:
* Name*
* Author
* Description*
* Date Created
* Last Updated
* User-uploaded title image*
* Tags (i.e. medieval,bridges)*
* List of schematics*
* Public/private*
* Downloads (direct + bundle)
* Rating
* Comments
* Flags

#####Creation
While browsing structures, users will be able to add them to their bundles on the fly. They can choose to use this feature for personal use (downloading a set of schematics), or to curate a bundle for the community. When creating a new bundle, users will be able to enter the information above marked with an asterisk(*). Tags for bundles should be plural and concerning a theme, rather than a list of the tags of the structures it contains. The user can also upload a banner image for their bundle.

#####Management
The user will need a list of their bundles. This will need to show bundle status, as well as if it has been published. From this list, the user can view a more detailed version of each bundle and edit them.

#####Technical
I believe bundles will just be zipped folders of the structures they contain, along with a metadata file containing some information for the settings menu.

###Quality Control
The structure library is useless without quality control. Schematic and bundle popularity will be driven by ratings and download counts. Addtionally, both will be able to be flagged as NSFW, incomplete, or not meeting quality standards. These flags will show up when viewing the schematic/bundle.

##6. Development and Timeline
In terms of development, I am by no means a pro Minecraft modder, but I have some experience. I made a fully functional cut/copy/paste/import/export/fill/replace mod, only to later figure out that Bukkit's WorldEdit existed. Either way, that gave me experience with a large portion of the modding process.

I won't be able to commit a ton of time to this project, but I still want to get started on it. I would happy and willing to work with others who are passionate about this project with the following skills: Java/Minecraft mod development, web development (full stack), graphic design. If you have any of those skills, or a skill you feel would be useful to the development of this mod, please contact me at [hollenbach.andrew@gmail.com](mailto:hollenbach.andrew@gmail.com). In terms of hosting, I will be able to host the web interface for free during the mod's early lifecycle. If the mod catches on, we will need to migrate to a more scalable solution. Additionally, I'm willing to purchase a domain once the mod title has been finalized. In terms of source, I am committed to keeping this project open source and under a free license like GPL.

###Key Resources (Tentative)
* Forge ModLoader
* Schematic file format
* ASP.NET MVC 5
* WebGL/JavaScript (for schematic file rendering)
* MySQL
* Git / GitHub

###Timeline
This is the order in terms of critical systems and priority.
#####Key Features
1. Create conversion tool to make Expand-compatible schematics from regular schematics
2. Successfully import schematics into Minecraft (from default bundle)
3. Place schematics in proper biome/height range and match anchors
5. Density adjustments
6. Online Structure Library (schematics only)
7. Add bundles to library

#####Desired Features
1. Schematic toggling (until this point, users can add/remove using file system) via settings menu

Obviously, some of those are much more specific than others. When those steps arrive, we'll have to set more specific plans of attack. I also might be forgetting a few features that I mentioned before.

##7. Conclusion
I'm looking forward to actually getting started. I've been thinking about this project for awhile and it's nice to finally set out a loose plan. This is by no means a comprehensive or technical document - at some points I am oddly specific, and other parts I am very vague, either because I haven't figured out a good way to do it, or it's just a simple task that doesn't need explanation. I do have to say this - the more developers get interested an involved, the sooner this will be done. I'm really excited!
