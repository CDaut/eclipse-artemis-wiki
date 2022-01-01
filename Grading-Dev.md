# Eclipse-Plugin for Grading with the [Artemis Project](https://github.com/ls1intum/Artemis)

## Development

### Setting up Eclipse
1. You should use "eclipse for eclipse commiters" for PlugIn development. This will make a lot of things easier.

2. Use the `docs/workingTargetDefinition.target` to create the target platform needed for this project.

    - Drag and drop the `docs/workingTargetDefinition.target` file into eclipse and hit the "reload target platform" link in the top right corner.
    - Select "OSGi - REST" as the target platform and reload again.
3. API Baseline
    - Navigate to "Window > Preferences > Plug-in Development > API Baselines" (This will only be available in the Eclipse for eclipse comitters build).
    - Click "Add Baseline...".
    - Select "A target platform", then click "Next".
    - Enter a name for the target platform.
    - As a target platform you will want to use the "OSGi - REST" platform. Select it and hit "Finish".
    - Click "Apply". Eclipse now builds something. This might take some time.
    - Click "Apply and close".
4. Open project
    - Clone this repository into your eclipse workspace.
    - Use File > Open Project from Filesystem to open the project.
    - Hit the "Directory..." button and select the repository cloned two steps ago. Now some (While writing this 13) modules should show up below.
    - Hit "Select All" (which might not actually select all) and then "Finish".
    - You now have the project imported. You might see some errors. The errors for the "jvm" module can be ignored. The project builds fine anyways.
5. Create a run configuration
    - Navigate to `edu.kit.kastel.sdq.eclipse.grading.client` and open the `plugin.xml` file.
    - Click the small green arrow in the top right corner
    - This will either build the PlugIn (you are done) or it will crash (you have to continue with the next step).
6. Fix run configuration
    - There is an issue where some dependencies are duplicated which prevents the PlugIn from building. 
    - To resolve this edit the run configuration by clicking "Run > Run Configurations" and selecting the tab "Eclipse Application > Eclipse Application"
    - Navigate to the Plug-ins tab.
    - Change the "Launch with" dropdown to "plug-ins selected below only". A long list of plugins will appear.
    - Hit "Apply" and then "Run". This will most likely not work though
    - Fix 1: (Fast but unlikely to work. Try this first.)
        - Click the "Select All" button on the right and then apply and run again.
        - This has worked on some machines, not all though
    - Fix 2: (Tedious but worked more often)
        - Click the "Validate Plug-ins" button.
        - A long lost of errors appears. Expand the errors until you find one of the type "2 versions of singleton \<plugin name> exist".
        - This is a duplicate plugin.
        - Find the plugin in the plugin list above. There should be 2 entries for the name with different versions.
        - Deselect the plugin with that name and the **lower** version, leaving the newer one active.
        - Again, apply and run.
        - Repeat this for all plugins that exhibit this kind of error.
        - At some point you will be left with only about three issues that are of a different kind and another eclipse instance will start. At that point you are done and you can start developing the grading tool plugin.

(Tested on x86_64 arch linux, 1.1.22)


### Architecture
The architectural idea is based on having three plugins and an API plugin over which those plugins communicate.
That allows for more easily exchanging view, core/ Backend or client and also clearly defines borders, making parallel work easier and reducing coupling.

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/architecture.png" alt="Backend state machine" width="600"/>

#### Core/ Backend

Our Backend (core package) provides functionality for

* managing annotations
* calculating penalties
* serializing and deserializing annotations (via artemis client as a network interface)
* mapping plugin-internal state to artemis-internal state
* keeping track of state

#### Artemis Client

The Artemis Client provides certain calls to artemis needed by the Backend.

#### GUI

### Creating new view elements
New view elements (buttons, tabs, etc.) should be added to the *ArtemisGradingView* class. 
Every tab has got his own method (e.g *createBacklogTab(...)* ).

IMPORTANT:

If the new view element is state-dependent, you have to update the state machine and add it to the set in the view. Otherwise it will not be updated during the *updateState()* method. Use:

```java
	// replace YOUR_TRANSITION with the certain transition
	this.addControlToPossibleActions(yourNewViewElement, Transition.YOUR_TRANSITION);
```

### New calls to the Backend

New calls to the Backend can be realized through the *ArtemisViewController* class. Then call the method in the view using the *ArtemisViewController*. 

When the class is getting to messy, it would be a good idea to separate the calls according to the Backend controllers 

### Changing Preferences

The preference page is defined in the *ArtemisGradingPreferencesPage* class. 
A new field can be added in the *createFieldEditors()* method. 
The initial values are set in the *PreferenceInitializer* class.

An example with the field for the absolute config path:

```java
	public void createFieldEditors() {
		this.absoluteConfigPath = new FileFieldEditor(PreferenceConstants.ABSOLUTE_CONFIG_PATH,
				"Absolute config path: ", this.getFieldEditorParent());
		this.addField(this.absoluteConfigPath);
	}
```

### Adding marker attributes

A new attribute to the marker can be added in the plugin.xml. If the field should appear in the Assessment Annotation View, a class needs to be created for the field and it must be added to the *markerSupportGenerator* in the plugin.xml. 

To make the name of the attribute easy to change, it should be defined as constant in the *AssessmentUtilities* class. The attribute should be set in the *addAssessmentAnnotaion(...)* method and the *createMarkerForAnnotation(...)* method in the *ArtemisViewController* class.

For examples just look in the plugin.xml at the *org.eclipse.ui.ide.markerSupport* extension and the *edu.kit.kastel.eclipse.grading.view.marker* package for the field classes.


### Creating a new PenaltyRule

1. Add a Class derived from *edu.kit.kastel.sdq.eclipse.grading.core.model.PenaltyRule*
2. Add a Constructor for that class in *edu.kit.kastel.sdq.eclipse.grading.core.config.PenaltyRuleDeserializer.PenaltyRuleType*.
    Note that herein, you have access to the penaltyRule's JsonNode, so you may fetch values you define in your config to construct your PenaltyRule:
```java
    public enum PenaltyRuleType {
            //Need to add a new enum value with a short Name (that must be used in the config file) and a constructor based on the json node.
            THRESHOLD_PENALTY_RULE_TYPE (ThresholdPenaltyRule.SHORT_NAME, penaltyRuleNode -> new ThresholdPenaltyRule(penaltyRuleNode.get("threshold").asInt(), penaltyRuleNode.get("penalty").asDouble())),
            CUSTOM_PENALTY_RULE_TYPE (CustomPenaltyRule.SHORT_NAME, penaltyRuleNode -> new CustomPenaltyRule()),
            MY_NEW_PENALTY_RULE_TYPE (MyNewPenaltyRule.SHORT_NAME, penaltyRuleNode) -> new MyNewPenaltyRule(...));
```
3. use the new PenaltyRule in your config.json:
```json
        "mistakeTypes" : [
            ...,
            {
                "shortName": "idk",
                "button": "MyMistakeType",
                "message": "You made a grave mistake.",
                "penaltyRule": {
                    "shortName": "myNewPenaltyRule",
                    "penalty": 5,
                    "penaltyOnMoreThanThreshold": 500,
                    "threshold": 4
                },
                "appliesTo": "style"
            }
        ]
```

### Controllers
There are three Controllers:

* The AssessmentController controlls a single assessment in terms of managing annotations. It provides Methods like
    * *addAnnotation(..)*
    * *getAnnotations()*
    * *resetAndRestartAssessment()*
    * *...*
* The ArtemisController handles artemis-related stuff, including
    * managing Locks and *Feedbacks* which contain data gotten from locking a submission.
    * retrieving information about Courses, Submissions, Exercises, Exams, ... from the artemis client
    * starting, saving and submitting assessments.
* The SystemwideController holds and manages the Backend state.
  It acts as the main interface to the GUI.
  All calls relevant to our Backend state (see section about the Backend state machine) go through here.
  
### Backend State Machine

For keeping the Backend state sane and consistent, we use a state machine. That also allows for greying out buttons in the gui:
![Backend State Machine](https://github.com/kit-sdq/programming-lecture-eclipse-artemis/blob/main/docs/Zustandshaltung-Automat.png)

* On every state-modifying call to *edu.kit.kastel.sdq.eclipse.grading.core.SystemwideController* (represented by transitions (edges) in the state machine graph), the according transition is applied in the state machine. If it isn't possible, the transition is not applied and the GUI is notified.
* Transitions are designed to represent button clicks in the GUI, which in turn are mapped to one Backend call each.
* Each "transition class" (represented by its name) has **one single** *next_state* and multiple *from_state*s.
* In every state, it is known, which transitions are allowed. These are retrieved via ISystemwideController::getCurrentlyPossibleTransitions() by the GUI.
* In certain situations, a state is changed but then reverted. This is not done across calls, so the GUI does not notice it.
