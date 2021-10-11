+++
title = "Workflow"
+++

### Creating a Workflow
The workflow component gives you an object oriented way to define a process or a life cycle that your object goes through. 
Each step or stage in the process is called a place. You do also define transitions that describe the action to get from one place to another.

A set of places and transitions creates a definition. A workflow needs a Definition and a way to write the states to the objects 
(i.e. an instance of a com.lemric.workflow.markingstore.MarkingStoreInterface).

Consider the following example for a blog post. A post can have one of a number of predefined statuses (draft, reviewed, rejected, published). 
In a workflow, these statuses are called places. You can define the workflow like this:

```java
import com.labudzinski.workflow.DefinitionBuilder;
import com.labudzinski.workflow.Definition;
import com.labudzinski.workflow.PlaceInterface;
import com.labudzinski.workflow.Transition;
import com.labudzinski.workflow.Workflow;
import com.lemric.workflow.markingstore.MethodMarkingStore;

import java.util.HashMap;
import java.util.ArrayList;
class Application {
    public static void main(String[] args) {
        DefinitionBuilder definitionBuilder = new DefinitionBuilder();
        Map<String, PlaceInterface> places = new HashMap<>() {{
            put("draft", new Place("draft"));
            put("reviewed", new Place("reviewed"));
            put("rejected", new Place("rejected"));
            put("published", new Place("published"));
        }};
        definitionBuilder.addPlaces(places);
        // Transitions are defined with a unique name, an origin place and a destination place
        definitionBuilder.addTransition(new ArrayList<Transition>() {{
            add(new Transition("to_review", new ArrayList<PlaceInterface>() {{
                add(places.get("draft"));
            }}, new ArrayList<PlaceInterface>() {{
                add(places.get("reviewed"));
            }}));
            add(new Transition("publish", new ArrayList<PlaceInterface>() {{
                add(places.get("reviewed"));
            }}, new ArrayList<PlaceInterface>() {{
                add(places.get("published"));
            }}));
            add(new Transition("reject", new ArrayList<PlaceInterface>() {{
                add(places.get("reviewed"));
            }}, new ArrayList<PlaceInterface>() {{
                add(places.get("rejected"));
            }}));
        }});

        Definition definition = definitionBuilder.build();

        Boolean singleState = true; // true if the subject can be in only one state at a given time
        String property = "currentState"; // subject property name where the state is stored
        MethodMarkingStore marking = new MethodMarkingStore(singleState, property);
        Workflow workflow = new Workflow(definition, marking);
    }
}
```