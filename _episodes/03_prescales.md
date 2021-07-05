---
title: "Getting trigger prescales"
teaching: 0
exercises: 25
questions:
- "How do I get the prescales for my triggers?"
objectives:
- "Learn how to extract the prescales for you triggers"
keypoints:
- "Prescales can be extracted using the trigger tools in CMSSW and/or from the [brilcalc](http://opendata.cern.ch/docs/cms-guide-luminosity-calculation#calculate-trigger-prescales-for-a-run-and-trigger-path) tool (not covered here)."
---

## The TriggerInfoTool

The CERN Open Data portal maintains some records that explain how to use certain tools for analysis.  One of those is the record about the [TriggerInfoTool](http://opendata.cern.ch/record/5004).  It eventually points to the [TriggerInfoTool repository](https://github.com/cms-opendata-analyses/TriggerInfoTool/tree/2011) on Github.  This is a place where you can find a few examples and explore the way the code is used for extracting trigger information.

## Getting the Prescales and the acceptance bit

Let's work on one of the examples stored in the repository above.  In particular, we are going to look at the [TriggerSimplePrescalesAnalyzer](https://github.com/cms-opendata-analyses/TriggerInfoTool/tree/2011/TriggerSimplePrescalesAnalyzer) example (package).  The following directions can also be found in that repository.

First make sure you are at the top of your `CMSSW_5_3_32/src` area and that you have issued the `cmsenv` command.

Now let's clone the *2011* branch of this repository, which will be good for the 2012 data we are working with (except for a couple of changes):

```bash
git clone -b 2011 git://github.com/cms-legacydata-analyses/TriggerInfoTool.git
```

Compile:

```bash
scram b
```

Edit the config file to choose the triggers we are interested in (you can use `nano` or other editor).  Remember, we will exercise the `TriggerSimplePrescalesAnalyzer` example:

```bash
nano TriggerInfoTool/TriggerSimplePrescalesAnalyzer/python/simpleprescalesinfoanalyzer_cfg.py
```
Replace the `PoolSource` files with the ones we used in our last episode, replace the triggerPatterns parameter with a simpler trigger, like `"HLT_Mu12_v??"` (note the wildcard at the end `??`, so we can get the prescales for all versions of this trigger).  Also, **make absolutely sure you have access to the conditions database information needed for 2012, which is different than that for 2011**.  They look like:

~~~
#needed to cache the conditions data
process.load('Configuration.StandardSequences.FrontierConditions_GlobalTag_cff')
#Uncomment if using CVMFS file system for accessing conditions
#process.GlobalTag.connect = cms.string('sqlite_file:/cvmfs/cms-opendata-conddb.cern.ch/FT53_V21A_AN6_FULL.db')
process.GlobalTag.globaltag = 'FT53_V21A_AN6::All'
~~~
{: .language-python}


These lines, with `GlobaTag` in them, have to do with being able to read CMSSW database information.  We call this the [Conditions Data](http://opendata.cern.ch/docs/cms-guide-for-condition-database) as we may find values for calibration, alignment, trigger prescales, etc., in there .  One can think of the `GlobalTag` as a label that contains a set of database snapshots that need to be adequate for a point in time in the history of the CMS detector.  For 2012 open data release, the global tag is `FT53_V21A_AN6` (the `::All` string is a flag that tells the frameworks to read *All* the information associated with the tag).  

The `connect` variable in one of those lines just modifies they way in which the framework is going to access these snapshots. Here, there is a key difference between using the **Virtual Machine** or the **Docker container**.  When using the Virtual Machine, all three lines need to be **uncommented**. This is because when using VMs, we read these conditions from the shared files system area at CERN (CVMFS), so we need it active.  Read in this way, the conditions will be cached locally in your virtual machine the first time you run and so the CMSSW job will be slow.  Fortunately, we already did this while setting up our VM, so our jobs will run much faster.  This does not really matter for the Docker container.

If you are using the Docker container, the connect line (second line of the group) needs to be **commented out**.

Feel free to just replace the whole config file with the example below and make the one change about the connection line we talked about.  

> ## Take a look at the config file
>
> The config file should look like:
> ~~~
> import FWCore.ParameterSet.Config as cms
>
> process = cms.Process("TriggerInfo")
>
> process.load("FWCore.MessageService.MessageLogger_cfi")
> #if more events are activated, choose to print every 1000:
> #process.MessageLogger.cerr.FwkReport.reportEvery = 1000
>
> process.maxEvents = cms.untracked.PSet( input = cms.untracked.int32(100) )
>
> process.source = cms.Source("PoolSource",
>     fileNames = cms.untracked.vstring(
>         'root://eospublic.cern.ch//eos/opendata/cms/Run2012B/TauPlusX/AOD/22Jan2013-v1/20000/0040CF04-8E74-E211-AD0C-00266CFFA344.root',
>         'root://eospublic.cern.ch//eos/opendata/cms/Run2012C/TauPlusX/AOD/22Jan2013-v1/310001/0EF85C5C-A787-E211-AFC9-003048C6942A.root'
>     )
> )
>
> #needed to get the actual prescale values used from the global tag
> process.load('Configuration.StandardSequences.FrontierConditions_GlobalTag_cff')
> #Uncomment the line below if using the VM
> #process.GlobalTag.connect = cms.string('sqlite_file:/cvmfs/cms-opendata-conddb.cern.ch/FT53_V21A_AN6_FULL.db')
> process.GlobalTag.globaltag = 'FT53_V21A_AN6::All'
>
> #configure the analyzer
> #inspired by https://github.com/cms-sw/cmssw/blob/CMSSW_5_3_X/HLTrigger/HLTfilters/interface/HLTHighLevel.h
> process.gettriggerinfo = cms.EDAnalyzer('TriggerSimplePrescalesAnalyzer',
>                               processName = cms.string("HLT"),
>                              triggerPatterns = cms.vstring("HLT_Mu12_v??"), #if left empty, all triggers will run
>                               triggerResults = cms.InputTag("TriggerResults","","HLT"),
>                               triggerEvent   = cms.InputTag("hltTriggerSummaryAOD","","HLT")
>                               )
>
>
> process.triggerinfo = cms.Path(process.gettriggerinfo)
> process.schedule = cms.Schedule(process.triggerinfo)
> ~~~
> {: .language-python}
{: .solution}

Let's run:

```bash
cmsRun TriggerInfoTool/TriggerSimplePrescalesAnalyzer/python/simpleprescalesinfoanalyzer_cfg.py  > full_prescales.log 2>&1 &
```

> ## Let's check the output
>
> ~~~
> HLTConfig has changed . . .
> Begin processing the 1st record. Run 194075, Event 14880766, LumiSection 48 at 29-Sep-2020 03:11:45.700 CEST
> Currently analyzing trigger HLT_Mu12_v16
> analyzeSimplePrescales: path HLT_Mu12_v16 [121] prescales L1T,HLT: 50,30
>  Trigger path status: WasRun=1 Accept=0 Error =0
> Begin processing the 2nd record. Run 194075, Event 14844046, LumiSection 48 at 29-Sep-2020 03:11:45.712 CEST
> Currently analyzing trigger HLT_Mu12_v16
> analyzeSimplePrescales: path HLT_Mu12_v16 [121] prescales L1T,HLT: 50,30
>  Trigger path status: WasRun=1 Accept=0 Error =0
> ...
> ~~~
> {: .output}
>
> Notice that the L1 prescale was 50 for that particular run, whereas the HLT prescale was 30.  We would have to run on many more events to see an `Accept=1`, which would mean the event was accepted by this trigger.  
>
{: .solution}

> ## Challenge!
>
> How about now you run with the triggers we were interested in, i.e., the `HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v?` ones.  What do you get?
>
> > ## Solution and work assignment
> >
<!--
> > You will see an output like this:
> > ~~~
> > HLTConfig has changed . . .
> > Begin processing the 1st record. Run 194075, Event 14880766, LumiSection 48 at 29-Sep-2020 02:26:23.553 CEST
> > Currently analyzing trigger HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2
> > %MSG-e HLTConfigData:  TriggerSimplePrescalesAnalyzer:gettriggerinfo  29-Sep-2020 02:26:23 CEST Run: 194075 Event: 14880766
> >  Error in determining L1T prescale for HLT path: 'HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2' with L1T seed: 'L1_SingleMu14er OR L1_SingleMu16er' using L1GtUtils: error code: 210001. > > (Note: only a single L1T name, not a bit number, is allowed as seed for a proper determination of the L1T prescale!)
> > %MSG
> > analyzeSimplePrescales: path HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2 [394] prescales L1T,HLT: -1,1
> >  Trigger path status: WasRun=1 Accept=0 Error =0
> > Begin processing the 2nd record. Run 194075, Event 14844046, LumiSection 48 at 29-Sep-2020 02:26:23.584 CEST
> > Currently analyzing trigger HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2
> > %MSG-e HLTConfigData:  TriggerSimplePrescalesAnalyzer:gettriggerinfo  29-Sep-2020 02:26:23 CEST Run: 194075 Event: 14844046
> >  Error in determining L1T prescale for HLT path: 'HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2' with L1T seed: 'L1_SingleMu14er OR L1_SingleMu16er' using L1GtUtils: error code: 210001. > > (Note: only a single L1T name, not a bit number, is allowed as seed for a proper determination of the L1T prescale!)
> > %MSG
> > ~~~
> > {: .output}
-->
> >
> > Life is not so easy sometimes.  While there was no problem getting the HLT prescale (it seems to be 1), i.e., the trigger is unprescaled at the HLT, we can't say
> > anything about the L1 because there is a limitation in our software.  You will get an error message.  Please copy this error message and paste it into the corresponding section in our [assignment form](https://forms.gle/DDboG1MCcSNRBRHFA); remember you must sign in and <strong style="color: red;">click on the submit button</strong> in order to save your work.  You can go back to edit the form at any time.
> >
<!--
> > ~~~
> >  Error in determining L1T prescale for HLT path: 'HLT_IsoMu17_eta2p1_LooseIsoPFTau20_v2' with L1T seed: 'L1_SingleMu14er OR L1_SingleMu16er' using L1GtUtils: error code: 210001. > > (Note: only a single L1T name, not a bit number, is allowed as seed for a proper determination of the L1T prescale!)
> > ~~~
> > {: .error}
-->
> >
> > Fortunately, there is a backup solution for us to check on the prescales.  Details will not be covered here, but you can find more information in [this guide](http://opendata.cern.ch/docs/cms-guide-luminosity-calculation#calculate-trigger-prescales-for-a-run-and-trigger-path) of the CODP.
> {: .solution}
{: .challenge}
> ## Explore the TriggerSimplePrescalesAnalyzer!
>
> On overtime, you could explore the implementation of the code we just used.  Play around with it, what else can you learn?
{: .discussion}



{% include links.md %}
