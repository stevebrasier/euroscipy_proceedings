:author: Steve Brasier
:email: steve.brasier@atkinsglobal.com
:institution: Atkins, 500 Park Avenue, Aztec West, BS32 4RZ 

:author: Fred Pollard
:email: fred.pollard@atkinsglobal.com
:institution: Atkins, 500 Park Avenue, Aztec West, BS32 4RZ 

------------------------------------------------------------
A Python-based Post-processing Toolset For Seismic Analyses
------------------------------------------------------------

.. class:: abstract

    This paper discusses the design and implementation of a Python-based
    toolset to aid in assessing the response of the UK's Advanced Gas
    Reactor nuclear power stations to earthquakes. The seismic analyses
    themselves are carried out with a commercial Finite Element solver, but
    understanding the raw model output this produces requires customised
    post-processing and visualisation tools. Extending the existing tools had
    become increasingly difficult and a decision was made to develop a new,
    Python-based toolset. This comprises of a post-processing framework
    (``aftershock``) which includes an embedded Python interpreter, and a
    plotting package (``afterplot``) based on ``numpy`` and ``matplotlib``.

    The new toolset had to be significantly more flexible and easier to
    maintain than the existing code-base, while allowing the majority of 
    development to be carried out by engineers with little training in software 
    development. The resulting architecture will be described with a focus on 
    exploring how the design drivers were met and the successes and challenges 
    arising from the choices made.

.. class:: keywords

   python, numpy, matplotlib, seismic analysis, plotting

Introduction
------------

Nuclear power in the UK is provided by a fleet of Advanced Gas-cooled Reactors (AGRs) which became operational in the 1970's. These are a second generation reactor design and have a core consisting of layers of interlocking graphite bricks which act to slow neutrons from the fuel to sustain the fission reaction. Although the UK does not regularly experience significant earthquakes it is still necessary to demonstrate that the reactors could be safely shut-down if a severe earthquake were to occur.

The response of the graphite core to an earthquake is extremely complex and a series of computer models have been developed to simulate the behaviour. These models are regularly upgraded and extended as the cores change over their lives to ensure that the relevant behaviours are included. The models are analysed using the commercial Finite Element Analysis code LS-DYNA. This provides predicted positions and velocities for the thousands of graphite bricks in the core during the simulated earthquake.

By itself this raw model output is not particularly informative, and a complex set of post-processing calculations is required to help engineers to assess aspects such as:

- Can the control rods still enter the core?
- Is the integrity of the fuel maintained?

This post-processing converts the raw position and velocity data produced by the model into parameters describing the seismic performance of the core, assesses these parameters against acceptable limits, and presents the results in tabular or graphical form.

This paper describes a recent complete re-write of this post-processing toolset. It seeks to explore some of the software and architectural decisions made and examine the impact of these decisions on the engineering users.

Background
----------

The LS-DYNA solver produces about 120GB of binary-format data for each simulation, split across multiple files. The existing post-processing tool was based on Microsoft Excel, using code written in Visual Basic for Applications (VBA) to decode the binary data and carry out the required calculations and Excel's graphing capabilities to plot the results. The original design of the VBA code was not particularly modular and its complexity had grown significantly as additional post-processing calculations were included and to accommodate developments in the models themselves. In short, there was significant "technical debt" [Cun92]_ in the code which made it difficult to determine whether new functionality would adversely impact the existing calculations.

The start of a new analysis campaign forced a reappraisal of the existing approach as these issues meant there was low confidence that the new post-processing features required could be developed in the time or budget available. The following requirements were identified as strongly desirable in any new post-processing tool:

- A far more modular and easily extensible architecture.
- More flexible plotting capabilities.
- A high-level, modern language to describe the actual post-processing calculations; these would be implemented by seismic engineers.
- Better performance; the Excel/VBA post-processor could take 4-6 hours to complete which was inconvenient.
- Possibility of moving to a Linux platform later, although starting initial development on Windows; this would allow post-processing to be carried out on a future Linux analysis server to streamline the work-flow and allow access to more powerful hardware.

A re-write from scratch would clearly be a major undertaking and was considered with some trepidation and refactoring the existing code would have been a more palatable first step. However further investigation convinced us that this would not progress a significant distance towards the above goals as the Excel/VBA platform was simply too limiting.

Overall Architecture
--------------------

An initial feasibility study lead to an architecture with three distinct parts:

#. A central C++ core, ``aftershock``, which handles the binary I/O and contains an embedded Python 2.7 interpreter.
#. A set of Python "calculation scripts" which define the actual post-processing calculations to be carried out.
#. A purpose-made Python plotting package ``afterplot`` which is based on ``matplotlib`` [Hun07]_.

As the entire binary dataset is too large to fit in memory at once the ``aftershock`` core operates frame-by-frame, stepping time-wise through the data. At each frame it decodes the raw binary data and calls defined functions from the calculation scripts which have been loaded. These scripts access the data for the frame through a simple API provided by ``aftershock`` which returns lists of floats. The actual post-processing calculations defined by the scripts generally make heavy use of the ``ndarrays`` provided by ``numpy`` [Wal11]_ to carry out efficient element-wise operations. As well as decoding the binary data and maintaining the necessary state for the scripts from frame-to-frame, the ``aftershock`` core also optimises the order in which the results files are processed to minimise the number of passes required.

The split between ``afterplot`` and a set of calculation scripts results in an architecture which:

a. Has sufficient performance to handle large amounts of binary data.
b. Has a core which can be reused across all models and analyses.
c. Provides the required high-level language for "users", i.e. the seismic engineers defining the calculations.
d. Hides the complex binary file-format entirely from the users.
e. Enforces modularity, separating the post-processing into individual scripts which cannot impact each other.

With Python selected as the calculation scripting language a number of plotting packages immediately became options. However ``matplotlib`` [Hun07]_ stood out for its wide use, "*publication quality figures*" [Hun07]_ and the sheer variety and flexibility of plotting capabilities it provided. Development of the post-processing toolset could have ended at this point, leaving the script engineers to utilise ``matplotlib`` directly. However ``matplotlib``\'s versatility comes with a price in complexity and the API is not particularly intuitive; requiring seismic engineers to learn the details of this did not seem to represent good value for the client. It was therefore decided to wrap ``matplotlib`` in a package ``afterplot`` to provide a custom set of very focussed plot formats.

Plotting Architecture
---------------------
``afterplot`` provides plotting functionality via a set of plotter classes, with the user (i.e. the engineer writing a calculation script) creating an instance of the appropriate class to generate a plot. All plotter classes inherit from a ``BasePlot`` class. This base class is essentially a wrapper for a ``matplotlib`` ``Figure`` object which represents a single plotting window, plus the ``Axes`` objects which represent the plots or sub-plots this contains.

At present ``afterplot`` provides only four types of plotter, although these are expected to be sufficient for most current requirements:

#. ``LayerPlot`` (Figure :ref:`LayerPlot`): This represents values on a horizontal slice through the model using a contour-type plot but using discrete markers.
#. ``ChannelPlot`` (Figure :ref:`ChannelPlot`): This represents the 3D geometry of a vertical column in the model by projection onto X-Z and Y-Z planes.
#. ``TimePlot`` (Figure :ref:`TimePlot`): This is a conventional X-Y plot, representing time-histories as individual series with time on the X-axis.
#. ``WaterfallPlot`` (Figure :ref:`WfallPlot`): This provides an overview of the distribution of the plotted parameter at each time-step during a simulation.

.. figure:: reld_Y_value_rr_1(2).png
   :scale: 40%
   :figclass: bht

   Example LayerPlot output :label:`LayerPlot`

.. figure:: channel-gui-3-trimmed.png
   :scale: 30%
   :figclass: bht

   Example ChannelPlot with GUI :label:`ChannelPlot`

.. figure:: tplot-nogui-1.png
   :scale: 30%
   :figclass: bht

   Example TimePlot output :label:`TimePlot`

.. figure:: wfall-nogui-3.png
   :scale: 40%
   :figclass: bht

   Example WaterfallPlot output :label:`WfallPlot`

Inherently all post-processed results are associated with a three-dimensional position within the model and a time within the simulation. Some parameters or outputs may collapse one or more of these dimensions, for example if plotting a plan view of peak values through time, maximums are taken over the vertical and time axes creating a set of results with two dimensions. All plotter classes therefore accept ``numpy`` arrays with up to four dimensions (or ``axes`` in numpy terminology). The meanings and order of these dimensions are standardised as three spatial dimensions followed by time, i.e. ``(x, y, z, t)``, so that different "views" of the same data can easily be generated by passing an array to different plotters.

Quality Advantages
------------------
A key advantage of providing a custom plotting package is that best-practice can be enforced on the generated plots, such as the provision of titles or use of grid-lines. Another example is that ``afterplot`` provides a custom   diverging colourmap as the default colourmap, based on the comprehensive discussion and methods presented in [Mor09]_. This should be significantly easier to interpret than the default colourmap provided by ``matplotlib`` in most cases.

The plotter classes can also allow *alteration of presentation*, e.g. axis limits, while preventing *modification of data*. Alteration of presentation is provided for by instance methods or GUI controls defined by the plotter classes. Modification of data is prevented simply by the lack of any interface to do this once the relevant array has been passed to the plot instance. This immutability is not intended as a security feature but simplifies quality assurance by limiting where errors can be introduced when altering presentation.

A further quality assurance feature is the capture of traceability data. When a new plot is generated, the ``BasePlot`` class traverses the stack frames using the standard library's ``inspect`` module to gather information about the paths and versions of calculation scripts and other Python modules used. This data is attached to the plots to assist in reproducing published plots or debugging issues. The use of introspection to capture this data means that this feature does not require any action by the script author.

Interactive GUI
---------------
Providing a simple GUI was considered desirable to bridge the gap for users from the previous Excel-based toolset. The ``matplotlib`` documentation describes two methods of providing a GUI:

1. Using the cross-backend widgets provided in ``matplotlib.widgets``, which are fairly limited.
2. Embedding the ``matplotlib.FigureCanvas`` object directly into the window provided by a specific GUI toolset such as ``Tk``.

An alternative approach is used by ``afterplot`` which is simpler than the second approach but allows the use of the richer widgets provided by specific GUI toolsets. This approach uses the ``pyplot.figure()`` function to handle all of the initial set-up of the GUI, with additional widgets then inserted using the GUI toolset's manager. This is demonstrated below by adding a ``Tk`` button to a ``Figure`` object using the ``TkAgg`` backend:

.. code-block:: python

    import Tkinter as Tk
    import matplotlib
    matplotlib.use('TkAgg')
    from matplotlib import pyplot
    class Plotter(object):
      def _init__(self):
        self.figure = pyplot.figure()
        window = self.figure.canvas.manager.window
        btn_next = Tk.Button(master=window,
                             text='next',
                             command=self._next)
        btn_next.pack(side=Tk.LEFT)
        self.figure.show()

Store and Restore
-----------------
Functionality to save plots to disk as images is provided by ``matplotlib`` via ``Figure.savefig()`` which can generate a variety of formats. When development of ``afterplot`` began a ``matplotlib.Figure`` object could not be pickled and therefore there was no native way to regenerate it for interactive use except for re-running the script which created it. Despite the improved performance provided by ``aftershock`` this is clearly time-consuming when only minor presentation changes are required such as altering the limits on an axis. A means to enable an entire plotter instance, including its GUI, to be stored to disk and later restored to a new fully interactive GUI was therefore strongly desirable. While the ability to pickle ``Figure`` objects has since been added to ``matplotlib`` this would not support the custom GUIs which ``afterplot`` provides. However, by following the same approach that the ``pickle`` module uses internally to handle class instances the desired store/restore functionality could be added relatively simply.


**Storing:**

#. When a plot instance is created, the ``__new__`` method of the ``BasePlot`` superclass binds the  supplied ``*args`` and ``**kwargs`` to attributes on the plotter instance - these will include one or more ``ndarrays`` containing the actual data to be plotted.
#. To store the instance, first a ``type`` object is obtained, then this and the ``*args`` and ``**kwargs`` are pickled.

Simplified code for the ``BasePlot`` class implementing storing to a given path:

.. code-block:: python

    class BasePlot(object):
      def __new__(cls, *args, **kwargs):
        obj = object.__new__(cls)
        obj._args, obj._kwargs = args, kwargs
        return obj
      def store(self, path):
        data = (type(self), self._args, self._kwargs)
        with open(path, 'w') as pkl:
          pickle.dump(data, pkl)
        def show(self):
          # .. gui code here ..

**Restoring**:

#. The type object, ``args`` and ``kwargs`` are unpickled from the file.
#. The type object is called to create a new instance, passing it the unpickled ``args`` and ``kwargs``.

Simplified restoring code, taking a path to a stored file and regenerating the plot complete with interactive GUI:

.. code-block:: python

    def restore(path):
      with open(path, 'r') as pkl:
        t_plt, args, kwargs = pickle.load(pkl)
        restored_plotter = t_plt(*args, **kwargs)
        restored_plotter.show()

Note that classes can define ``__getstate__`` and ``__setstate__`` methods to control how they are pickled and un-pickled and the approach described above could be implemented in these methods. The use of explicitly-named methods and functions was primarily to make the approach transparent to future developers.

This approach has a number of benefits:

1. Neither the storing nor restoring code needs to know anything about the actual plot class, except that it has a ``show()`` method, hence any plotter derived from ``BasePlot`` inherits this functionality.
2. The only interface which storing and restoring needs to address is the plotter class's signature. This is simple and robust, as code can always be added to a class's ``__init__`` method to handle changes in the signature such as depreciated parameters, meaning that it should essentially always be possible to make stored plots forward-compatible with later versions of ``afterplot``. By contrast normally when a class instance is unpickled, ``pickle`` directly sets the instance's dictionary (the ``__dict__`` attribute) from the pickled data, meaning that changes to an instance's internal attributes can break unpickling.
3. Restoring the interactive GUI requires no additional code - only what is needed to create the GUI when the plotter instance is first created.
4. If a stored plot is restored with a later version of ``afterplot`` any enhanced GUI functionality will automatically be available.

For convenience a simple ``cmd`` script and short Python function also allow stored plots to be restored on user's local Windows PCs and the GUI displayed by simply double-clicking the file. Alternatively a simple script can be written to batch process presentational changes such as colour bars or line thicknesses for a series of plots. Such a script uses a provided ``restore()`` function to restore the desired plots without showing the GUI, then uses the methods the plotter classes provide to alter desired presentation aspects.

One complication omitted from the simplified code above is that ideally storing and restoring should be insensitive to whether parameters have been specified as positional or named arguments. Therefore the ``__new__()`` method of the ``BasePlot`` superclass uses ``inspect.getargspec()`` to convert all arguments to a dictionary of ``name:value``. Class instances are then actually stored/restored as if all parameters were provided as keyword arguments.

While this approach essentially mirrors how ``pickle`` handles class instances, implementing such complex and robust
functionality in such little code is an impressive demonstration of Python’s benefits.

Outcomes and Lessons Learnt
---------------------------
The overall architecture has been a success:

- Performance is significantly improved.
- Post-processing can easily be integrated with analysis runs if required.
- Maintainability and extensibility of the calculations has been vastly improved.
- Python and ``numpy`` form a vastly more usable and concise high-language for describing calculations than VBA, allowing engineers to concentrate on the logic rather than working around the language.
- The ``aftershock`` core is reusable across different models which will save considerable effort for future models.
- Cross-platform portability to Windows and Linux was achieved without any significant effort for the calculation scripts and plotting module, making a decision to transition part-way through the project to new Linux hardware relatively straightforward.

However there were a number of challenges, some of which were expected at the outset and some which were not:

*Education and training:* As discussed a key driver for the architecture was that the calculation scripts would be written by seismic engineers, as they were the domain experts. Some of these engineers were already familiar with Python, often from scripting environments provided by commercial analysis software, or with other high-level scripting languages such as VBA. In general users found it relatively simple to pick up and start developing procedural and simple object-orientated Python, but the heavy use of ``numpy`` for element-wise operations then required users to learn a third programming paradigm. While the basic concepts were easily understood, deciding when the use of explicit loops or element-wise operations is more appropriate requires considerably more experience. Most engineers had not written code where performance was a concern and hence basic optimisation techniques such as moving constant expressions outside of loops were not necessarily considered obvious. Inconsistencies in the API for the scientific Python stack also led to some subtle performance and functionality issues; for example the three examples below all have different answers as to which package is "best":

- ``abs()`` vs. ``numpy.abs()``
- ``math.exp()`` vs. ``numpy.exp()``,
- ``math.pi`` vs. ``scipy.pi`` vs. ``numpy.pi``

*Development practicalities*: Some significant difficulties were encountered in compiling ``afterplot`` on both Windows and Linux due to the embedded Python 2.7 interpreter, but these issues are outside the scope of this paper to discuss.

*Plotting functionality:* The success of the ``afterplot`` plotting module is less clear at present. It has provided the desired plotting flexibility, as demonstrated by the ``LayerPlot`` and ``WaterfallPlot`` plot types which could not be easily replicated using Excel's plotting facilities. The control of style it enforces also appears to be strongly desirable in terms of reducing the effort required to obtain publication-quality plots. However verification of the relatively complex GUI code has proved to be difficult. "Verification" in this sense does not refer to a formal proof of correctness, but to a level of independent checking consistent with that applied to the actual post-processing calculations. Part of the difficulty with this was due to the limited internal availability of developers familiar with the GUI toolset. Another aspect was the decision to provide a small number of relatively general-purpose plot classes. This made it necessary for the plot classes to accept data in different dimensions and with a variety of options, complicating the internal logic which often involves complex array striding and reshaping. It may have been simpler overall to provide a larger number of less flexible plotters with simpler interfaces and fewer internal code paths. Testing plotting code is not straightforward but ``matplotlib``'\s own test suite has provided some useful techniques to automatically check images produced by test cases against known-good results. 

Overall, the decision to use the Python scientific software stack for this toolset has been strongly positive. Encouragingly it also appears that future develoments are likely to provide features like sparse arrays and lazy evaluation which would permit the calculation scripts to be simpler and more efficient. Similarly, rationalisation of the ``matplotlib`` API is expected in future which will simplify the creation of high-quality plots from Python.

References
----------

.. [Cun92] W Cunningham. *The WyCash Portfolio Management System*,
           OOPSLA '92 Addendum to the proceedings on object-oriented programming systems, languages, and applications, pp. 29-30.

.. [Wal11] S. Van Der Walt, S. Chris Colbert, Gaël Varoquaux. *The NumPy array: a structure for efficient numerical computation*,
           Computing in Science and Engineering, 13(2):22-30, 2011.

.. [Hun07] J. D. Hunter. *Matplotlib: A 2D Graphics Environment*,
	       Computing in Science & Engineering, 9(3):90-95, 2007.

.. [Mor09] K. Moreland. *Diverging Color Maps for Scientific Visualization*,
           Proceedings of the 5th International Symposium on Visual Computing, 2009.

