# Eclipse-Plugin for Grading with the [Artemis Project](https://github.com/ls1intum/Artemis)

## How do I run the plugin?

* Our GitHub CI run builds eclipse distros (Linux / Windows) containing the plugin. Download it [here](https://github.com/kit-sdq/programming-lecture-eclipse-artemis/actions/workflows/products.yml)
* In case you are using OS X, you might need to add our Update Site to run the plugin. It can be found [here](https://kit-sdq.github.io/programming-lecture-eclipse-artemis/)

### Working with the GUI
The GUI consists (mainly) of two parts:

* Artemis Grading View
* Assessment Annotations View

Both can be open directly by open the Artemis Perspective.

#### Artemis Grading View

The Artemis Grading View is a tab folder with three tabs:

* Grading
* Assessment
* Backlog

#### Grading

The grading tab can be used to create annotations for the downloaded submission. The tab is generated whenever a new submission is downloaded.
For creating a new annotation do the following:

* Mark the lines, where the annotation should be
* Click on the mistake button in the certain rating group
* After that you should see the new annotation in the assessment annotation view.

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/grading_tab.png" alt="Grading tab with the certain rating groups and mistake types" width="400" />

#### Assessment
The assessment tab has the following functions:

* Select course, exercise and exam
* Start assessment (correction round 1 or 2)
* Refresh Artemis state (if something changed)

After a new assessment is started, it is possible to reload, save or submit the started assessment (using the buttons).

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/assessment_tab.png" alt="Assessment tab" width="600"/>

#### Backlog

The backlog tab can be used to reload submissions that are already started, saved or submitted. One can filter the submission by selecting a filter in the first combo. 

It is important to selected the specified course and exercise in the assessment tab, otherwise the submission cannot be loaded. After a submission is loaded again, it can be graded and submitted normally. 

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/backlog_tab.png" alt="Backlog tab" width="600" />

#### Artemis Grading Preferences

User data are persisted in the Artemis Grading Preferences. You can what you need to fill in down below. The relative path of the config refers to the downloaded project and can be activated by the check box. To update the preferences, the "Refresh Artemis State" button must be pressed in the Assessment tab. Otherwise the changes will only take effect after a restart.

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/preferences.png" alt="Artemis Preferences" width="600" />

#### Assessment Annotation View

The assessment annotation view shows the annotations of the downloaded submission. 
The annotation can be deleted by deleting the marker. 

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/annotation_marker.png" alt="Two annotations and the corresponding view" width="600"/>

### Backend Configuration

#### Configuration File
To Configure mistake types, rating groups and whatnot, we use a config file.
See [https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/config_v4.json](docs/examples/config_v4.json) for an example configuration.

There are rating groups, mistake types and penalty rules.
The main config features are explained in the following.

#### Rating Groups
A rating group consists of multiple mistake types and an optional *penaltyLimit*. That limit is used for penalty calculation.
```json
"ratingGroups": [
    ...,
    {
        "shortName": "modelling",
        "displayName": "OO-Modellierung",
        "penaltyLimit": 16
    }
]
```

#### Mistake Types
A mistake type belongs to a rating group and has a penalty rule that defines the penalty calculation logic. Config File:
```json
"mistakeTypes" : [
    {
        "shortName": "custom",
        "button": "Custom Penalty",
        "message": "",
        "penaltyRule": {
            "shortName": "customPenalty"
        },
        "appliesTo": "style"
    },
    {
        "shortName": "jdEmpty",
        "button": "JavaDoc Leer",
        "message": "JavaDoc ist leer oder nicht vorhanden",
        "penaltyRule": {
            "shortName": "thresholdPenalty",
            "threshold": 1,
            "penalty": 5
        },
        "appliesTo": "style"
    }
]
```
See the Development chapter for more info about creating a new `PenaltyRule`.

#### Penalty Calculation/ Artemis Mapping

Currently, there are two penalty rule types you may use in your config:

* `ThresholdPenalty`: Iff the number of annotations with the given `mistakeType >= $threshold`, then `penalty` is added
* `CustomPenalty`: The tutor defines the message and the penalty.

Penalty Calculation is done rating-group-wise. For each rating group:

* all mistake types are evaluated: The corresponding annotations are used to calculate the mistake type's contribution.
* All mistake types' contributions are summed and optionally compared against the rating group's penalty limit which acts as a cap.
* Each annotation generates a MANUAL feedback, visible in the editor. No penalty points given, here!
* Each Rating group generates a `MANUAL_UNREFERENCED` feedback, visible *below* the editor (in the browser artemis client). Here, penalty points are given.
* Also, one (or more) MANUAL_UNREFERENCED feedback (invisible for students) is generated, which is used as a database for this client (containing serialized client-specific annotation data, including model identifiers, gui markers, startLine, endLine, ...)

