.. sit2standpy usage

=======================================
Usage examples
=======================================

Basic Use (New/v2 API)
----------------------

Basic usage of ``Sit2StandPy`` to detect transitions in the sample data, using the version 2 API

.. code-block:: python

    >>> import sit2standpy as s2s

    >>> # transform the data into the appropriate format for H5 or dictionary
    >>> # note that "data_transform_function" is your own function to achieve the
    >>> # appropriate format
    >>> # if you are looking for a quick example data loader function, you can
    >>> # use the one at
    >>> # https://gist.github.com/LukasAdamowicz/b8481ef32e4beeb77c80f29f34c8045e
    >>> data = <data_transform/loader_function>(acceleration_data)
    >>>
    >>> sequence = s2s.v2.Sequential()
    >>> # window the data into days using only the hours from 8:00 to 20:00
    >>> sequence.add(s2s.v2.WindowDays(hours=[8, 20]))
    >>> # Do the initial filtering and processing required
    >>> sequence.add(s2s.v2.AccelerationFilter())
    >>> # Detect the transitions using the stillness constraint
    >>> sequence.add(s2s.v2.Detector(stillness_constraint=True))
    >>>
    >>> sequence.predict(data)  # predict and save the results into data
    >>>
    >>> # tabulate the results to a csv for easy reading
    >>> s2s.v2.tabulate_results(data, path_to_csv_output, method='stillness')


Basic Use (Old/v1 API)
----------------------

Basic usage of ``Sit2StandPy`` to detect transitions in sample data:

.. code-block:: python

    >>> import sit2standpy as s2s
    >>> import numpy as np  # importing sample data
    >>> from sys import version_info
    >>> if version_info < (3, 7):
    >>>     from pkg_resources import resource_filename
    >>> else:
    >>>     from importlib import resources
    >>>
    >>> # locate the sample data and load it (depending on python version)
    >>> if version_info < (3, 7):
    >>>     file_path = resource_filename('sit2standpy', 'data/sample.csv')
    >>>     data = np.loadtxt(file_path, delimiter=',')
    >>> else:
    >>>     with resources.path('sit2standpy.data', 'sample.csv') as file_path:
    >>>         data = np.loadtxt(file_path, delimiter=',')
    >>>
    >>> # separate the stored sample data
    >>> time = data[:, 0]
    >>> accel = data[:, 1:]
    >>>
    >>> # initialize the framework for detection
    >>> ths = {'stand displacement': 0.125, 'transition velocity': 0.3, 'accel moving avg': 0.15,
    >>>                    'accel moving std': 0.1, 'jerk moving avg': 2.5, 'jerk moving std': 3}
    >>> sts = s2s.Sit2Stand(method='stillness', gravity=9.84, thresholds=ths, long_still=0.3, still_window=0.3,
    >>>                     duration_factor=4, displacement_factor=0.6, lmin_kwargs={'height': -9.5}, power_band=[0, 0.5],
    >>>                     power_peak_kwargs={'distance': 128}, power_stdev_height=True)
    >>>
    >>> # run the sit-to-stand detection
    >>> SiSt = sts.apply(accel, time, time_units='us')
    >>>
    >>> # print the list of Transition objects, stored as a dictionary with the time they occurred
    >>> print(SiSt)

Advanced Examples (Old/v1 API)
------------------------------

Using :meth:`sit2standpy.Sit2Stand` automatically does all the preprocessing, filtering, and sit-to-stand transition
detection for the user. However, this can be broken up into the constituent parts - preprocessing, filtering, and
detection.

.. code-block:: python

    >>> import sit2standpy as s2s
    >>> from packages_for_importing_data import your_import_data_function
    >>>
    >>> # due to the size of multi day files, no samples are provided with sit2standpy
    >>> accel, time = your_import_data_function()
    >>>
    >>> # PREPROCESSING : conversion of timestamps, and windowing the data
    >>> timestamps, dt, acc_win = s2s.process_timestamps(time, accel,
    >>>                                                  time_units='us',  # time is in microseconds since the epoch
    >>>                                                  window=True,  # window the data
    >>>                                                  hours=('08:00', '20:00'))  # use data from 8am to 8pm each day
    >>>
    >>> # setup filter
    >>> afilt = s2s.AccelerationFilter(power_band=[0, 0.5], power_peak_kw={'distance': 128})
    >>> # filter the windowed data, iterating over the days
    >>> filt_acc, rm_acc, power_peaks = {}, {}, {}
    >>> for day in acc_win.keys():  # dictionary of data for each day, since data was windowed
    >>>     filt_acc[day], rm_acc[day], _, power_peaks[day] = afilt.apply(acc_win[day], 1 / dt)
    >>>
    >>> # setup the detection of the transitions
    >>> still_detect = s2s.detectors.Stillness()  # use the default values
    >>>
    >>> sist = {}
    >>> for day in filt_acc.keys():
    >>>     day_sist = still_detect.apply(acc_win[day], filt_acc[day], rm_acc[day], timestamps[day], dt,
    >>>                                   power_peaks[day])
    >>>     sist.update(day_sist)
