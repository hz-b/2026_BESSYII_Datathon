# Datathon Challenge: Convert myspot experiment data in to NeXus standand format

- [x] Step 1: Before you start - Explore the data!
- [x] Step 2: Prepare json mapping file
- [x] Step 3: Make use of eln (Explore the capability of structured eln data)
- [x] Step 4: Run converter
- [x] Step 5: Visualize (preferably in NOMAD)


# Revision of pynxtools from Day 1:

The input parameters are defined as follows:

**reader**: The specific reader which gets called inside pynxtools. This is supplied in the pynxtools python code. If you create a specific reader for your measurement file it gets selecetd here. **We will use built-in json_map reader for this challlenge.**

**nxdl**: The specific nxdl file to be used. For fluoroscence measurement [`NXfluo`](https://manual.nexusformat.org/classes/applications/NXfluo.html) differaction measurement [`NXmonopd`](https://manual.nexusformat.org/classes/applications/NXmonopd.html).
    
**configuration**: json file which stores the mapping to NeXus data structure. 

**ELN**: yaml file which can be used to store additional (meta)data to NeXus data structure. 

**output**: The output filename of the NeXus file.

