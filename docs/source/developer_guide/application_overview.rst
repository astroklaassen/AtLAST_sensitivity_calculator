Application overview
====================
The AtLAST sensitivity calculator consists of a Python package and a web front-end that calculates either the achievable sensitivity or required integration time given an integration time or sensitivity (respectively).
An overview of each component is provided below.

The calculator
--------------

The ``atlast_sc`` Python package contains the code that performs sensitivity or
integration time calculations based on input parameters provided by the user, and from those inputs, it derives parameters required for the calculation. The code identifies allowed values and units for the parameters and performs validation of inputs provided to the calculator. It
also provides utility tools for reading input data from a file and writing output
to file.

Information on using the Python package is provided :doc:`here <../user_guide/using_the_calculator>`.

Modules
^^^^^^^
Below is an overview description of each of the modules included in the
``atlast_sc`` package. More detailed information is provided in the
:doc:`Public API <../code_docs/public_api>` and :doc:`UML diagrams <../code_docs/uml>`
sections.

calculator_factory
++++++++++++++++++
This module creates a calculator instance according to user input that has been specified. 

calculator
++++++++++
This module contains the main ``Calculator`` class that provides the interface
for performing sensitivity and integration time calculations. A ``Calculator``
object may be instantiated with default parameter setup object, or by passing
user input parameters as arguments to the parameter setup object constructor.

This module provides methods exclusively for calculating sensitivity and integration time.
It retrieves parameter sets —including user inputs, telescope and environmental 
conditions, and derived parameters— through the parameter setup object. This design 
simplifies the process of setting up a  calculation for users by providing a unified interface to access information from each parameter class.

parameter_setup
+++++++++++++++
This class serves as a centralised container for all parameter classes. When a parameter is 
updated in its respective class, the new value can be retrieved through this class. It 
provides access to all models used in calculations and methods for operations related to 
identifying applicable instruments. The class also stores a copy of the
parameters used to initialize the calculator, allowing the user to revert to
the initial state.

data
++++
The ``Data`` class stores all of the configuration information for each of the user input, 
telescope and environment parameters used by the calculator (default values, default units, etc.), 
and calculated parameters.

The ``Validator`` class provides methods for validating data provided to the calculator.

models
++++++
This module contains model definitions that describe the structure of the data
provided to the calculator. The module uses the ``pydantic`` library; models
within the module inherit from the pydantic ``BaseModel``. Custom validation methods
within the models ensure that input data is of the right type and satisfies the
constraints defined in the ``data.Data`` class.

derived_groups
++++++++++++++
This module contains classes that logically group derived parameters used by
the calculator. Derived parameters are those that are dependent on the data
provided to the calculator (both as user input and internally specified or derived telescope and environment parameters). They are calculated at runtime when the calculator is instantiated, and when any of the independent parameters are updated.

The derived group classes are ``AtmosphereParams``, ``Efficiencies``, and
``Temperatures``. Although these classes are accessible via the public API, they
are primarily intended to be used internal to the calculator.

exceptions
++++++++++
This module contains the data validation exception and warning classes.

utils
+++++
This is a utility module that contains classes and methods used throughout the application.

Class Structure
^^^^^^^^^^^^^^^
General class structure can be visualised with the UML diagrams below.

.. image:: imgs/calculator_class.png
    :alt: Diagram of relation between Calculator and CalculatorFactory class
    :align: center

The above diagram shows how the CalculatorFactory class has the Calculator class as a dependency. 
The below diagram shows how each of the parameter classes depend on each other and how the 
ParameterSetup class acts as the container for the current state of each parameter class.

.. image:: imgs/parameter_classes.png
    :alt: Diagram of parameter classes that make up the calculation process
    :align: center

Integration Overview
--------------------
The overall calculation process is begins with a creation of a Calculator object using  CalculatorFactory. At starup, the calculator instance is created with default values. If the calculator is used via the Python CLI, any of the user input parameters can be changed before calculating the sensitivity/integration time. If the input parameters are not changed, the calculations will be done with initialised default values. In the UI, the first calculation is done with the default values and any specified user input parameters will be considered within the calculations once the user clicks the "Calculate" button. 

Based on user inputs, the application will choose an instrument configuration to use for calculating sensitivity or integration time. The user can override the instrument choice manually, and the code will check whether the chosen instrument is applicable to the input calculation parameters. Currently, only the CLI users are able to choose a specific 
instrument to use in their calculations. For more details about the instrument selection 
process refer to the :ref:`Instrument Selection <instrument selection>` section.

Once the instrument has been selected, based on either calculator or user selection, the calculator will use the instrument specific equations and parameters to use when calculating sensitivity/integration time. These instrument specific equations and parameters are defined within the respective instrument YAML files and classes. 

.. _instrument selection:

Instrument Selection
--------------------
Instrument selection in the web UI is done in the backend when a validated set of input parameters are sent for calculation via the "Calculate" button. The backend compares the user input observing frequency and bandwidth to the supported ranges for each instrument and chooses the correct instrument to use given those inputs. In the case where the user input parameters correspond to more than one instrument, the calculator will choose the first applicable instrument. If there are no applicable instruments, the calculator will proceed with the Default instrument. 

However, on the CLI, the user can make an instrument selection, and explicitly change the calculator chosen instrumet. Because input validation and subsequent parameter derivation happens on input change, the calculator will reject instrument selections that do not meet current observing frequency and bandwidth selections. As such, any user input parameter should be specified before selecting an instrument. For example, if the user 
wants to set a specific observing frequency to do calculations and also select an instrument, 
they have to make sure that the observing frequency they are specifying falls into the
observing frequency ranges of the instrument they want to select. They should also take
care to do the same with the bandwidth values. In the case where the user attempts to 
select an instrument before specifying the appropriate observing frequency and bandwidth 
values, the calculator will throw an error. 

The applicable observing frequency and bandwidth ranges for each instrument along with some
other information can be accessed by listing the instruments on the CLI. 

Adding a new instrument
^^^^^^^^^^^^^^^^^^^^^^^

To add a new instrument to the calculator there are 2 files that need to be created in the  *atlast_sc/instruments* directory, and the overall ``config.py`` file in that directory needs updating to look for those files. The first is a YAML file that should be named after the instrument and should be created in the *data* sub-directory. The second is the python file, also named after the instrument, to be placed in the *classes* sub-directory.

Creating the instrument YAML file
+++++++++++++++++++++++++++++++++


The YAML file in the *data* directory contains the parameter space over which the instrument setups are valid, and any other information needed to do sensitivity calculations in the python file. An example is provided below: 

.. code-block:: yaml

    name: "Example"
    allowed_ranges:
        observing_frequency:
            ranges: [(500.0-600.0),(700.0-800.0)]
            unit: GHz
        bandwidth: 
            ranges: [(10.9e4-1.8e8)]
            unit: Hz
    receiver_temperature: 
        values: [30.0,40.0]
        unit: K

Any other instrument specific parameters should be added following the same data formatting model. In the code, the 'Default' instrument YAML file could be taken as a template, with the other instrument files serving as examples of how to expand the file to fit the needs of new instruments.

Creating the instrument Python module
+++++++++++++++++++++++++++++++++++++
A Python module file, named after the instrument should be created in the *classes* sub-directory. Consistent with the YAML example above, the name of the module file should be "Example.py" and it should include the following class format: 

.. code-block:: python 

    """
    Example instrument parameters
    """        
    class Example(Instrument):
        def __init__(self, data):
            super().__init__(data)

The 'Default' instrument Python module can be taken as an example of how to setup the instrument module files, with the other instrument Python module files showing how that default can be modified to reflect new instrumentation. 

Modifying the configuration file to see the new instrument
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Once those two files are in place, ``config.py`` needs to be updated with information about the names and locations of those files in the initialisation method, and then adding that dictionary to the ``available_instruments`` list. In the initilisation 
method, a dictionary containing pointers to the new instrument's Python module 
and YAML file name should be added with the same formatting as the existing instruments. 


The web application
-------------------
The web client consists of a backend based on the `FastAPI web framework <https://fastapi.tiangolo.com/lo/>`__,
a standard, browser-based HTML/CSS/JavaScript frontend, and a REST API.

The FastAPI application renders the frontend using
the `Jinja templating engine <https://jinja.palletsprojects.com/en/3.1.x/>`__.

FastAPI also auto-generates an OpenAPI schema that can be used to render interactive,
browser-based documentation of the REST API. The documentation can be accessed via the following two URLs:

- ``<app_url>/docs`` to render with Swagger UI
- ``<app_url>/redoc`` to render with Redoc

where ``<app_url>`` is the root URL of the application (e.g., ``localhost:8000``).
