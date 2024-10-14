+++
title = 'Arquillian: Scenario Generator'
date = 2021-09-02
draft = false
+++

Arquillian is a powerful testing framework for Java that allows developers to test their code against Java containers. In other words, Arquillian is able to package an ad-hoc archive and deploy it into a specified Java container. As a consequence, integration tests become much easier to set up and handle. Moreover, Arquillian is also integrated with famous testing frameworks, i.e. JUnit and TestNG, which allows developers to run integration tests with usual building tools like Maven and Ant. For more information about Arquillian and its powerful features, please visit the project’s homepage [1] or refer to the introductory book of John D. Ament [2].

In this blog post, a tricky testing scenario is addressed using advanced features of Arquillian: the design of an integration test simulating a Server/Client architecture. The two roles of this distributed model will be played by two applications, which will be deployed to two different Java containers. A test suite takes the role of the Client and it will be _always_ deployed using Arquillian; a servlet application takes the role of the Server but Arquillian will not always be the responsible party to start the deployment: the servlet can be externally deployed and, therefore, Arquillian will be required to skip its deployment routine, _**which is the tricky part of this testing scenario**_. To summarise, this blog post will discuss how to run the following scenarios as part of the same testing routine:
* Scenario 1 – Arquillian manages both Java containers, one hosting the servlet and the other one hosting the test suite. In this scenario, Arquillian will deploy both the servlet and the test suite
* Scenario 2 – Arquillian manages only the Java container that hosts the test suite. As a consequence, the deployment of the test suite is handled by Arquillian. An external building tool (e.g. Maven) is responsible to deploy the servlet to an external runtime environment (e.g. Quarkus) or a Java container

{{< figure src="arquillian_scenario_generator.svg" class="m-auto mt-6" >}}

When the above scenarios are run as part of the same testing suite, Arquillian’s standard API cannot be employed to handle the situation. In fact, the annotation _@Deployment_ deploys the generated _ShrinkWrap_ artifacts at every run, thus, it does not address the need to dynamically deploy the servlet on-demand. Luckly, the Arquillian’s SPI comes in handy when customised deployment logics need to be created. In particular, in this blog post an Arquillian extension based on _DeploymentScenarioGenerator_, will be used to deploy the servlet application on-demand. It is worth noting that this solution is also compatible with _unmanaged_ Arquillian containers. Moreover, using _DeploymentScenarioGenerator_ allows Arquillian to properly handle exceptions thrown during the test.

Refer to [1] for more detailed explanations of any of the concepts discussed above. The next section will illustrate how the plumbing of the integration with the test suite works in practice.

First of all, the code to create the Arquillian extension will be introduced and partially explained. The code presented here is currently used in the leading open source transaction manager Narayana [3]. For future reference, the Pull Request that has introduced this design change is #1871 and the branch in the forked repository is [4]. Following, only the parts of the Narayana code base related to the design of the Arquillian extension will be discussed.

#### LRACoordinatorScenarioGenerator
```java
public class LRACoordinatorScenarioGenerator extends ScenarioGeneratorBase
        implements DeploymentScenarioGenerator {

    public static final String EXTENSION_NAME = "LRACoordinatorDeployment";
    public static final String EXTENSION_DEPLOYMENT_NAME = "deploymentName";
    public static final String EXTENSION_GROUP_NAME = "groupName";
    public static final String EXTENSION_CONTAINER_NAME = "containerName";
    public static final String EXTENSION_TESTABLE = "testable";

    @Override
    public List<DeploymentDescription> generate(TestClass testClass) {

        List<DeploymentDescription> descriptions = new ArrayList<>();

        // Fetch all properties in the section EXTENSION_NAME
        Map<String, String> extensionProperties = getExtensionProperties(EXTENSION_NAME);

        /*
         * If the section of this extension is not in the arquillian.xml file,
         * it means that this extension does not need to start.
         * As a consequence, an empty list of DeploymentDescription is returned
         */
        if (extensionProperties == null) {
            return new ArrayList<>();
        }

        // Checks that all required properties are defined
        checkPropertiesExistence(
                extensionProperties,
                EXTENSION_DEPLOYMENT_NAME,
                EXTENSION_GROUP_NAME,
                EXTENSION_CONTAINER_NAME,
                EXTENSION_TESTABLE);

        GroupDef group = getGroupWithName(extensionProperties.getOrDefault(EXTENSION_GROUP_NAME, ""));

        ContainerDef container;
        if (group != null) {
            container = getContainerWithName(group, extensionProperties.get(EXTENSION_CONTAINER_NAME));
        } else {
            container = getContainerWithName(extensionProperties.get(EXTENSION_CONTAINER_NAME));
        }

        if (container == null) {
            String message = String.format(
                    "%s: no container was found with name: %s.",
                    EXTENSION_NAME,
                    extensionProperties.get(EXTENSION_CONTAINER_NAME));

            log.error(message);
            throw new RuntimeException(message);
        }

        String containerName = container.getContainerName();

        WebArchive archive = (WebArchive) new WildflyLRACoordinatorDeployment()
                .create(extensionProperties.get(EXTENSION_DEPLOYMENT_NAME));

        DeploymentDescription deploymentDescription =
                new DeploymentDescription(extensionProperties.get(EXTENSION_DEPLOYMENT_NAME), archive)
                        .setTarget(new TargetDescription(containerName));
        deploymentDescription.shouldBeTestable(Boolean.parseBoolean(extensionProperties
                .get(EXTENSION_TESTABLE)));
        // Auto-define if the deployment should be managed or unmanaged
        deploymentDescription.shouldBeManaged(!container.getMode().equals("manual"));

        descriptions.add(deploymentDescription);

        return descriptions;
    }
}
```

Along the _DeploymentScenarioGenerator_ interface, the above class also extends the class _ScenarioGeneratorBase_, which is a base class to collect all helper methods to access and process information in _arquillian.xml_. This is a crucial point in the design of this extension: a new custom section is introduced in _arquillian.xml_ and it is used as a switch to turn on and off the deployment of the servlet.

#### arquillian-wildfly.xml
```xml
<extension qualifier="LRACoordinatorDeployment">
    <property name="groupName">${arquillian.group.qualifier}</property>
    <property name="containerName">${arquillian.lra.coordinator.container.qualifier}</property>
    <property name="deploymentName">${arquillian.lra.coordinator.deployment.qualifier}</property>
    <property name="testable">false</property>
</extension>
```

In the same custom section, new properties can also be defined to pass extra info to the Arquillian extension (e.g., the Java container’s name, the preference to deploy a testable artifact or not, the cluster where the container is, etc.). Thanks to the extra information in arquillian.xml, the developed Arquillian extension can tailor an ad-hoc artifact that can be deployed on-demand. This new feature addresses the initial problem presented above. In fact, when an additional arquillian.xml is defined without specifying the custom section to activate the Arquillian extension, two runs of the same test routing can be started and both scenario 1 and scenario 2 can be tested.

_NOTE_. The deployment method that creates the actual ShrinkWrap artifact can also be referenced using reflection instead of hardcoding the name of the method in the Arquillian extension; this would allow to specify the deployment function directly in * and would make the extension even more dynamic.

Once the code of the extension is in place and activated specifying the extension’s fully qualified name in `src/test/java/resources/META-INF.services/org.jboss.arquillian.core.spi.LoadableExtension`, the last step left is to define arquillian.xml according to the properties specified in the extension.

### References
1. Arquillian.org. Arquillian – Write Real Tests. [online] Available at: <https://arquillian.org/> [Accessed 2 September 2021]
1. Ament, J., 2013. Arquillian testing guide. Birmingham, UK: Packt Pub
1. Narayana.io. Narayana. [online] Available at: <https://narayana.io/> [Accessed 2 September 2021]
1. jmfinelli, 2021. GitHub – jmfinelli/narayana at JBTM-3444. [online] GitHub. Available at: <https://github.com/jmfinelli/narayana/tree/JBTM-3444> [Accessed 2 September 2021].

