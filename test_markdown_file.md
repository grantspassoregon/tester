# test
# Updating Web Services

For redundancy, we host our published GIS services both in our Enterprise portal as well as AGOL.
Services published to our Enterprise portal are referenced data, meaning that as we update our database, these changes will be reflected in the published service in near real-time.
However, the services published to AGOL are copied onto their servers, and will become out-of-date when additional changes are made to our database.
AGOL is often the preferred provider, because we are leveraging their server resources to deliver better performance, so the majority of services on our public-facing web viewer are reading from AGOL.
However, this also means we must periodically update the content on AGOL, so that the latest data is available to our users.

Here we delineate the steps to ensure a complete sync between our database and AGOL:

## Step One: Reconcile Versions in the City Editor

When city staff submit edits to a layer through the internal web viewer, they are modifying the DRAFT version of the database.
Likewise, the ArcGIS projects that we use to make edits are also pointed at the DRAFT version.
The version of our data shared with the public on our web viewer is the PRODUCTION/DEFAULT version.
In order for changes made in DRAFT to be visible to the public, we must first push these changes into the PRODUCTION version.

- You can reconcile versions in any ArcPro project by selecting the List by Data Source tab in the Contents pane, and clicking on the versioned layer.
- A Version tab will appear in the menu ribbon, and selecting this tab will reveal a set of versioning options in the ribbon.
- Click the Reconcile button (typically you will target the DEFAULT version, defining conflict by column and resolving conflicts in favor of the edit version).
- When the reconcile has completed, use the Version Changes button to view the results.
- When the version changes pane opens, select the Differences tab to view specific changes that will be pushed into production.
- Click Post to push the changes to the PRODUCTION version.

Review and approve changes before merging. If the changes are edits that you have personally made, the review process may be relatively brief. But if the changes are submitted by field crews and other staff, take a moment to inspect the differences and ensure that they are reasonable and well-formed. Defer to the expertise of field crews for establishing facts on the ground, but keep in mind they may need help to enter the data idiomatically, using the correct fields and domain values.

## Step Two: Push Utility Edits to Beehive

When edits include changes to any utility layer, we also need to push these changes to the Beehive mapping service that we use to for event management. Utility workers use the Beehive map to track recurring maintenance and record progress on their tasks, so from their perspective, if it has not been uploaded to Beehive, the update may as well have not occurred. Currently, we use the _beehive_push.mdx_ project in ArcMap to update the Beehive layer. This can be found in the _GIS General\Public_Works_Projects directory_.

- Select all updated features in a layer.
- Select the layer of interest in the contents pane.
- Click the Beehive Update button.

Per Mark Janz, I push point assets first, then line assets.

## Step Three: Stage Draft Services

Here we use a script that depends on methods in the `arcpy` module to stage draft services in a folder, so we can upload a new version of the service to AGOL.
To access `arcpy`, we have to use the version of Python installed with ArcGIS.
This is not the same as the version of Python we use for normal development, so we effectively have two versions of Python on our machine.
Since I have my IDE pointed at the latest/greatest version on my local machine, the script will fail to compile if I run it from my IDE, with a complaint that it cannot find the `arcpy` module.
I am sure there is a project-specific way to point the IDE at the correct Python version, but I have not explored this option.
Instead, when I need to run a script with `arcpy`, I run it from inside ArcPro.
In this case, still using the _city_editor_ project, I open the python window and enter the commands:

```{bash}
exec(open('c:/users/erose/repos/scripts/service_draft.py').read())
drafts.draft_services(base_path)
```

The first command opens the script located at the path location and runs the contents. This creates an object called _drafts_, containing methods for creating and staging the service drafts, including the _draft_services_ method that we call here. The _base_path_ variable is set to a folder on the O: drive. This folder contains the previous set of drafts, and gets overwritten with every call to this script. This is the same directory that we will read from when publishing the services in the next step.

The script _service_draft.py_ includes logging messages, but these will not show up in ArcPro. We can pipe log messages into `arcpy`, but this is a different logging workflow from the normal python _logging_ library, and I have not had a chance to learn it. Instead, I just want to read the logging messages that are already in there. To do that, open a terminal window and enter the command:

```{bash}
Get-Content P:/service_update.log -Wait -Tail 0
```

If you have updated the location of the log file in _service_draft.py_, then you will need to update the path here.
The terminal will stay open and print any additions to the log file.
You can exit the listening state by pressing CTRL-C.
If you open the log file in a text editor, you can also read the contents, but the contents will not update upon new entries to the log, so I prefer watching the file from the terminal.
The main issue to watch out for are failures, cases where the service draft could not be staged.
Currently the output looks like this:

```{bash}
09/30/2024 04:05:37 PM Exception ERROR 160069: The dataset was not found.
Failed to execute (StageService).
09/30/2024 04:05:37 PM Dropping planning.
```

In the above example, the _planning_ service failed to update automatically. We will need to open the project file for this service and determine why it is failing to build. When the staging succeeds, the message will read "Draft staged for X". If you have made adjustments to a service, and wish to try staging the draft again for that service in particular, you can pass the service name in a vector to the _draft_services_ method:

```{bash}
drafts.draft_service(base_path, ["planning"])
```

## Step Four: Update Web Services

The _service_update.py_ script uses the `arcgis` module to publish the draft services to AGOL. The `arcgis` module is an open-source Python package, unlike `arcpy`, so we can run it from the terminal using a Python virtual environment, as touched on [here](./virtual_env.md).

From within your virtual environment, navigate to the _scripts_ directory and enter the command:

```{bash}
exec(open('login.py').read())
```

This will open a window in your browser asking you to approve the login. Click Approve, copy the SAML code and paste it into the terminal using CTRL-SHIFT-V. Once you have successfully authenticated, enter the command:

```{bash}
exec(open('service_update.py').read())
```

When the script runs, it will print the following:

```{bash}
Run: services.publish(gis, base_path, short)
```

The _services_ object holds methods to read the contents of the service drafts directory and upload the services to AGOL, including the _publish_ method invoked here.
The _gis_ variable holds the AGOL portal connection, and _base_path_, as before, points to the directory containing the service drafts.
The _short_ variable contains the subset of services that receive frequent updates. For services that do not update often, it is wasteful to rebuild the services if no changes have occurred.
As with the _draft_services_ method, you can substitute a vector of service names in place of _short_.
Indeed, if you print the contents of _short_, it shows you the vector of service names to update by "default":

```{python}
> short
['agreements', 'historic_cultural_areas', 'land_use', 'impervious_surface', 'merlin_landfill', 'parking', 'parks', 'planning', 'regulatory_boundaries', 'sewer_utilities', 'stormwater', 'transportation', 'water_utilities', 'zoning']
```

The services that typically fail to update are _historic_cultural_areas_, _water_utilities_, _sewer_utilities_, and _transportation_. The issue may be related to amount of data in the service, which may be leading to time-outs during the upload stage. For each service that fails to update by script, I open the ArcPro project associated with the service, and republish the service from there.
