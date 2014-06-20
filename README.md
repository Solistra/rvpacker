rvpacker
======================

A tool to unpack & pack RPGMaker data files into text so they can be version controlled & collaborated on

rvpacker consists of 3 parts:

* RPG library (stub classes for serialization of RPGMaker game data)
* RGSS library (some more classes for RPGMaker serialization)
* rvpacker (the script you call on the frontend)

Credit to SiCrane
=================

The RPG and RGSS libraries were originally taken from SiCrane's YAML importer/exporter on the gamedev forums. I initially just put them in github so I wouldn't lose them, and added the rvpacker script frontend. They are starting to drift a bit, but SiCrane still gets original credit for the grand majority of the work that rvpacker does.

http://www.gamedev.net/topic/646333-rpg-maker-vx-ace-data-conversion-utility/

Installation
============

rvpacker is bundled as a rubygem

    $ gem install rvpacker

... dependencies will be handled automatically.

**Windows Users:** If `gem install rvpacker` complains about being unable to install psych, download the rvpacker source, `gem install bundler`, and run `bundle install` from the project directory. That, for some reason, works. (Don't ask me, I think ruby is retarded here.) It is possible to install Psych 2.0.0, but the "stock" version lacks some [necessary YAML generation tweaks](https://github.com/akesterson/rvpacker/blob/master/README.md#psych-200-dependency).

Usage
=====

    $ rvpacker --help
    Options:
	    --action, -a <s>:   Action to perform on project (unpack|pack)
	   --project, -d <s>:   RPG Maker Project directory
		 --force, -f:   Update target even when source is older than target
      --project-type, -t <s>:   Project type (vx|ace|xp)
		  --help, -h:   Show this message

For example, to unpack a RPG Maker VX Ace project in ~/Documents/RPGVXAce/Project1:

    rvpacker --action unpack --project ~/Documents/RPGVXAce/Project1 --project-type ace

... This will expand all Data/* files into (PROJECT)/YAML/ as YAML files (YAML is used because the object serialization data is retained, which ruby's YAML parser is very good at - otherwise I would have changed it to JSON). The Scripts will be unpacked as individual .rb files into (PROJECT)/Scripts/.

To take a previously unpacked project, and pack it back up:

    rvpacker --action pack --project ~/Documents/RPGVXAce/Project1 --project-type ace

... This will take all of the yaml files in (PROJECT)/YAML and all the scripts in (PROJECT)/Scripts, and repack all of your (PROJECT)/Data/* files. You can trust this to completely reassemble your Data/ directory, so long as the Scripts/ and YAML/ directories remain intact.

Workflow : General
========

This is great for teams that are collaborating on an RPG Maker project. Just add a few steps to your existing workflow:

* Checkout the project from version control
* Run 'rvpacker --action pack' on the project to repack it for the RPG Maker tool
* Load up RPG Maker and do whatever you're going to do; save the project
* Run 'rvpacker --action unpack' on the project
* Commit everything to version control (ignore the Data directory since you don't need it anymore; use .gitignore or .hgignore or whatever)

... Now your project can be forked/merged in a much more safe/sane way, and you don't have to have someone bottlenecking the entire process.

Workflow : Avoiding Map Collisions
=======================

One thing that rvpacker really can't help you with right now (and, ironically, probably one of the reasons you want it) is map collisions. Consider this situation:

* The project has 10 maps in it, total.
* Developer A makes a new map. It gets saved by the editor as Map011
* Developer B makes a new map, in a different branch. It also gets saved by the editor as Map011.
* Developer A and Developer B attempt to merge their changes. The merge fails because of the collision on the Map011 file.

The best way to avoid this that I can see is to use blocks of pre-allocated maps. You appoint one person in your project to be principally responsible for the map assets. It then becomes this person's responsibility to allocate maps in "blocks", so that people can work on maps in a distributed way without clobbering one another. The workflow looks like this:

* The project has 10 maps in it, total.
* Developer A needs to make 4 maps. He sends a request to the "map owner", requesting a block of 4 maps.
* The map owner creates 4 default, blank maps, and names them all "Request #12345" for Developer A
* Developer A starts working on his maps
* Developer B needs to make 6 maps. He sends a request to the "map owner", requesting a block of 6 maps.
* The map owner 4 default, blank maps, and names them all "Request #12346" or something similar for Developer B
* Developer B starts working on his maps

Using this workflow, it doesn't matter what order Developers A and B request their map blocks, or what order the map owner creates their map blocks. By giving the map owner the authority to create the map blocks, the individual developers can work freely in their map blocks. They can rename them, reorder them, change all of the map attributes (Size, tileset, whatever), without getting in danger of a map collision.

While this may seem like unnecessary process, it is a reasonable workaround. For a better explanation of why rvpacker can't do this for you, read the next section.

Automatic ID generation
=======================

You can add new elements to the YAML files manually, and leave their 'id:' field set to 'null'. This will cause the rvpacker pack action to automatically assign them a new ID number at the end of the sequence (e.g., if you have 17 items, the new one becomes ID 18). This is mainly handy for adding new scripts to the project without having to open the RPG maker and paste the script in; just make the new script file, add its entry in YAML/Scripts.yaml, and the designer will have your script accessible the next time they repack and open the project.

Also, the rvpacker tool sets the ID of script files to an autoincrementing integer. The scripts exist in the database with a magic number that I can't recreate, and nothing in the editor (RPG VX Ace anyway) seems to care if the magic number changes. It doesn't even affect the ordering. So in order to support adding new scripts with null IDs, like everything else, the magic numbers on scripts are disregarded and a new ID number is forced on the scripts when the rvpacker pack action occurs.

Note that this does not apply to Map files. Do not try changing the map ID numbers manually (see the "Avoiding Map Collisions" workflow, above, and "Why rvpacker can't help with map collisions", below).

Why rvpacker can't help with map collisions
===========================================

If you look at the map collision problem described above, the way out of this situation might seem obvious: "Rename Map011.yaml in one of the branches to Map012.yaml, and problem solved. However, there are several significant problems with this approach:

* The ID numbers on the map files correspond to ID number entries in MapInfos.yaml (and the corresponding MapInfos.rvdata objects)
* The ID numbers are used to specify a parent/child relationship between one or more maps
* The ID numbers are used to specify the target of a map transition/warp event in event scripting

This means that changing the ID number assigned to a map (and, thereby, making it possible to merge 2 maps with the same ID number) becomes very nontrivial. The event scripting portion, especially, presents a difficult problem for rvpacker to overcome. It is simple enough for rvpacker to change the IDs of any new map created, and to change the reference to that ID number from any child maps. However, the events are where it gets sticky. The format of event calls in RPG Maker map files is not terribly well defined, and even if it was, I sincerely doubt that you want rvpacker tearing around in the guts of your map events.


Psych 2.0.0 Dependency
======================

From SiCrane:

I used cygwin's ruby 1.9.3 and the Psych 2.0.0 ruby gem, which appears to be the most recent version. However, Psych 2.0.0 has some bugs that impacted the generated YAML (one major and one minor) which I monkey patched, and since I was already rewriting the Psych code, I added some functionality to make the generated YAML prettier. Long story short, this code probably won't work with any version of Psych but 2.0.0.
