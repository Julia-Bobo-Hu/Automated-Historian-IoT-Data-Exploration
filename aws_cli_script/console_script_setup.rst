.. _console-script-setup:


AWS cli Script Setup
=================

Usage
------------
To use the AWS cli script to trigger launcher lambda function:

.. code-block:: bash

    aws lambda invoke --function-name REPLACE_EXAMPLE_FUNCTION_NAME --payload file://lambdaPayload.json --region us-east-1 lambdaOutput.txt

.. code-block:: bash

    python setup.py develop

.. code-block:: bash

    python setup.py install
    pip install mypackage