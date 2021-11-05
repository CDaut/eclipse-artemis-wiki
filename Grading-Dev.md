# Eclipse-Plugin for Grading with the [Artemis Project](https://github.com/ls1intum/Artemis)

## Development

### Setting up Eclipse

1. Use the docs/workingTargetDefinition.target to create the target platform needed for this project.
2. Adjust your Run Configuration accordingly ("Plugins->Select All" will do)

### Architecture
The architectural idea is based on having three plugins and an API plugin over which those plugins communicate.
That allows for more easily exchanging view, core/ Backend or client and also clearly defines borders, making parallel work easier and reducing coupling.

<img src="https://github.com/kit-sdq/programming-lecture-eclipse-artemis-grading/blob/main/docs/architecture.png" alt="Backend state machine" width="600"/>

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
![Backend State Machine](https://github.com/kit-sdq/programming-lecture-eclipse-artemis-grading/blob/main/docs/Zustandshaltung-Automat.png)

* On every state-modifying call to *edu.kit.kastel.sdq.eclipse.grading.core.SystemwideController* (represented by transitions (edges) in the state machine graph), the according transition is applied in the state machine. If it isn't possible, the transition is not applied and the GUI is notified.
* Transitions are designed to represent button clicks in the GUI, which in turn are mapped to one Backend call each.
* Each "transition class" (represented by its name) has **one single** *next_state* and multiple *from_state*s.
* In every state, it is known, which transitions are allowed. These are retrieved via ISystemwideController::getCurrentlyPossibleTransitions() by the GUI.
* In certain situations, a state is changed but then reverted. This is not done across calls, so the GUI does not notice it.
