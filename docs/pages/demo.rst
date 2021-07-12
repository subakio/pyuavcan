.. _demo:

데모
====

이 섹션에서 PyUAVCAN을 이용해서 `UAVCAN <https://uavcan.org>`_ 어플리케이션을 빌드하는 방법을 보여준다.
GNU/Linux와 Windows에서 테스트하였다; 다른 OS에서도 동작되리라 예상한다.
이 문서는 다음과 같은 순서로 구성되어 있다.:

- 첫번째 섹션에서 2개 커스텀 데이터 타입을 소개하고 어떻게 처리되는지 본다.

- 두번째 섹션에서 온도 제어기를 구현과 커스텀 RPC-서비스를 제공하는 간단한 데모 노드를 보여준다.

- 세번째 섹션에서 Yakut를 이용해서 UAVCAN의 데이터 분산 기능을 보여준다. --- (커맨드라인 유틸리티로 분석과 UAVCAN 네트워크 디버깅에 이용)

- 네번째 섹션에서 플랜트를 시뮬레이션하는 두번째 노드를 추가하고 온도를 첫번째 노드로 제어한다.

- 마지막 섹션에서 UAVCAN 네트워크의 동작 및 관리 설정 방법에 대해서 설명한다.

*UAVCAN node*, *DSDL*, *subject-ID*, *RPC-service*와 같은 용어에 익숙하리라 가정한다.
만약 익숙하지 않다면 먼저 <https://uavcan.org/guide>`_ 를 훑어보도록 하자.

다음을 따라할려면 :ref:`PyUAVCAN 설치 <installation>`와 계속하기 전에 새로운 디렉토리로 스위치하자.


DSDL 정의
----------------

모든 UAVCAN 어플리케이션은 네임스페이스 ``uavcan``에 위치하는 표준 DSDL 정의를 따른다.
표준 네임스페이스는 UAVCAN 프로젝트에서 유지하는 *규정된* 네임스페이스 부분이다.
git에서 복사하기::

    git clone https://github.com/UAVCAN/public_regulated_data_types

데모에서는 루트 네임스페이스 ``sirius_cyber_corp``에 위치한 2개 벤더 데이터 타입을 이용한다.
루트 네임스페이스 디렉토리 레이아웃은 다음과 같다::

    sirius_cyber_corp/                              # root namespace directory
        PerformLinearLeastSquaresFit.1.0.uavcan     # service type definition
        PointXY.1.0.uavcan                          # nested message type definition

Type ``sirius_cyber_corp.PerformLinearLeastSquaresFit.1.0``,
file ``sirius_cyber_corp/PerformLinearLeastSquaresFit.1.0.uavcan``:

.. literalinclude:: /../demo/custom_data_types/sirius_cyber_corp/PerformLinearLeastSquaresFit.1.0.uavcan
   :linenos:

Type ``sirius_cyber_corp.PointXY.1.0``,
file ``sirius_cyber_corp/PointXY.1.0.uavcan``:

.. literalinclude:: /../demo/custom_data_types/sirius_cyber_corp/PointXY.1.0.uavcan
   :linenos:


첫번재 노드
----------

아래 주어진 소스 코드를 복사하여 ``demo_app.py`` 파일 이름에 붙여넣기 한다.
명확히 하기 위해서 커스텀 DSDL 루트 네임스페이스 디렉토리 ``sirius_cyber_corp/`` 를 위에서 생성한 ``custom_data_types/``로 옮긴다.
이제 다음과 같은 디렉토리 구조가 된다.::

    custom_data_types/
        sirius_cyber_corp/                          # Created in the previous section
            PerformLinearLeastSquaresFit.1.0.uavcan
            PointXY.1.0.uavcan
    public_regulated_data_types/                    # Clone from git
        uavcan/                                     # The standard DSDL namespace
            ...
        ...
    demo_app.py                                     # The thermostat node script

여기에 ``demo_app.py``가 있다:

.. literalinclude:: /../demo/demo_app.py
   :linenos:

만약 스크립트를 실행한다면,
*missing registers*에 대한 에러가 나타나게 된다.

코멘트에서 설명한 것과 같이( --- 아주 상세하게 --- UAVCAN 스펙에서)
레지스터는 기본적으로 이름을 갖는 값으로 로컬 UAVCAN 노드의(어플리케이션) 여러 설정 파라미터를 유지하고 있다.
이런 파라미터의 일부는 어플리케이션의 비지니스 로직에서 사용된다.(예제 PID gains);
다른 파라미터는 UAVCAN 스택에서 사용된다.(port-IDs, node-ID, 트랜스포트 설정, 로깅 등등)
후반 카테고리에 있는 레지스터는 모두 동일한 접두어 ``uavcan.`` 이름을 가지고 있고 다른 이름과 의미는 에코시스템의 일관성을 위해서 스펙에서 규정하고 있다.

따라서 일부 어플리케이션은 정보를 읽어올 수 있는 레지스터가 없기 때문에 UAVCAN 네트워크에 도달할 수 없다는 에러를 발생시킨다.
환경 변수를 통해서 올바른 레지스터를 전달하면 이를 해결할 수 있다.:

..  code-block:: sh

    export UAVCAN__NODE__ID=42                           # Set the local node-ID 42 (anonymous by default)
    export UAVCAN__UDP__IFACE=127.9.0.0                  # Use UAVCAN/UDP transport via 127.9.0.42 (sic!)
    export UAVCAN__SUB__TEMPERATURE_SETPOINT__ID=2345    # Subject "temperature_setpoint"    on ID 2345
    export UAVCAN__SUB__TEMPERATURE_MEASUREMENT__ID=2346 # Subject "temperature_measurement" on ID 2346
    export UAVCAN__PUB__HEATER_VOLTAGE__ID=2347          # Subject "heater_voltage"          on ID 2347
    export UAVCAN__SRV__LEAST_SQUARES__ID=123            # Service "least_squares"           on ID 123
    export UAVCAN__DIAGNOSTIC__SEVERITY=2                # This is optional to enable logging via UAVCAN

    python demo_app.py                                   # Run the application!

sh/bash/zsh에서 동작한다.; 만약 Windows에서 PowerShell을 사용하는 경우에는 ``export`` 를 ``$env:``로 바꾸고 값을 따옴표 처리한다.

``UAVCAN__SUB__TEMPERATURE_SETPOINT__ID`` 환경 변수는 ``uavcan.sub.temperature_setpoint.id`` 레지스터를 설정한다.

PyUAVCAN에서 레지스터는 *register file*에 보통 저장되고 이 경우 ``my_registers.db``에 저장된다.(UAVCAN 스펙은 레지스터들이 어떻게 저장되어야 하는지에 대한 규정하지 않고 상세 구현에 대한 내용이다.)
일단 특정 설정으로 어플리케이션을 구동시키면 레지스터 파일에 값을 저장하며 다음에는 환경 변수를 전달하지 않고 이를 실행할 수 있다.

UAVCAN 노드의 레지스터는 표준 DSDL 네임스페이스 ``uavcan.register``에 정의된 표준 RPC-서비스를 통해서 다른 네트워크 참가자에게 노출된다. 다른 관리 인터페이스에 의존하지 않고도 가능하다.
이는 데모 어플리케이션과 같은 SW 노드와 임베디드 하드웨어 노드에 대해서 동일하다.

위에 명령을 실행하려면 실행되는 스크립트를 보게 된다.
실행되도록 남겨두고 다음 섹션으로 가보자.

..  tip:: Just-in-time vs. ahead-of-time DSDL compilation

    이 스크립트는 요청한 DSDL 네임스페이스를 소스 코드로 바로 변환한다.
    이런 접근법은 일부 어플리케이션에 적합하지만, 전반적으로 재배포를(예: PyPI를 통해) 위해서 만들어진 경우 DSDL ahead-of-time (빌드할때) 컴파일로부터 오는 장점이 있고 컴파일된 결과를 재배포 패키지로 포함한다.
    Ahead-of-time DSDL 컴파일은 ``setup.py``에서 구현할 수 있다:

    .. literalinclude:: /../demo/setup.py
       :linenos:


Yakut를 사용해서 node를 poking하기
---------------------------

The demo is running now so we can interact with it and see how it responds.
We could write another script for that using PyUAVCAN, but in this section we will instead use
`Yakut <https://github.com/UAVCAN/yakut>`_ --- a simple CLI tool for diagnostics and management of UAVCAN networks.
You will need to open a couple of new terminal sessions now.

If you don't have Yakut installed on your system yet, install it now by following its documentation.

Yakut requires us to compile our DSDL namespaces beforehand using ``yakut compile``:

.. code-block:: sh

    yakut compile  custom_data_types/sirius_cyber_corp  public_regulated_data_types/uavcan

The outputs will be stored in the current working directory.
If you decided to change the working directory or move the compilation outputs,
make sure to export the ``YAKUT_PATH`` environment variable pointing to the correct location.

The commands shown later need to operate on the same network as the demo.
Earlier we configured the demo to use UAVCAN/UDP via 127.9.0.42.
So, for Yakut, we can export this configuration to let it run on the same network anonymously:

..  code-block:: sh

    export UAVCAN__UDP__IFACE=127.9.0.0  # We don't export the node-ID, so it will remain anonymous.

To listen to the demo's heartbeat and diagnostics,
launch the following commands in new terminals and leave them running:

..  code-block:: sh

    export UAVCAN__UDP__IFACE=127.9.0.0
    yakut sub uavcan.node.Heartbeat.1.0     # You should see heartbeats being printed continuously.

..  code-block:: sh

    export UAVCAN__UDP__IFACE=127.9.0.0
    yakut sub uavcan.diagnostic.Record.1.1  # This one will not show anything yet -- read on.

Now let's see how the simple thermostat node is operating.
Launch another subscriber to see the published voltage command (it is not going to print anything yet):

..  code-block:: sh

    export UAVCAN__UDP__IFACE=127.9.0.0
    yakut sub -M 2347:uavcan.si.unit.voltage.Scalar.1.0     # Prints nothing.

And publish the setpoint along with measurement (process variable):

..  code-block:: sh

    export UAVCAN__UDP__IFACE=127.9.0.0
    export UAVCAN__NODE__ID=111         # We need a node-ID to publish messages
    yakut pub --count 10 2345:uavcan.si.unit.temperature.Scalar.1.0   'kelvin: 250' \
                         2346:uavcan.si.sample.temperature.Scalar.1.0 'kelvin: 240'

You should see the voltage subscriber that we just started print something along these lines:

..  code-block:: yaml

    ---
    2347: {volt: 1.1999999284744263}

    # And so on...

Okay, the thermostat is working.
If you change the setpoint (via subject-ID 2345) or measurement (via subject-ID 2346),
you will see the published command messages (subject-ID 2347) update accordingly.

One important feature of the register interface is that it allows one to monitor internal states of the application,
which is critical for debugging.
In some way it is similar to performance counters or tracing probes:

..  code-block:: sh

    yakut call 42 uavcan.register.Access.1.0 'name: {name: thermostat.error}'

We will see the current value of the temperature error registered by the thermostat:

..  code-block:: yaml

    ---
    384:
      timestamp: {microsecond: 0}
      mutable: false
      persistent: false
      value:
        real32:
          value: [10.0]

Field ``mutable: false`` says that this register cannot be modified and ``persistent: false`` says that
it is not committed to any persistent storage (like a register file).
Together they mean that the value is computed at runtime dynamically.

We can use the very same interface to query or modify the configuration parameters.
For example, we can change the PID gains of the thermostat:

..  code-block:: sh

    yakut call 42 uavcan.register.Access.1.0 '{name: {name: thermostat.pid.gains}, value: {integer8: {value: [2, 0, 0]}}}'

Which results in:

..  code-block:: yaml

    ---
    384:
      timestamp: {microsecond: 0}
      mutable: true
      persistent: true
      value:
        real64:
          value: [2.0, 0.0, 0.0]

An attentive reader would notice that the assigned value was of type ``integer8``, whereas the result is ``real64``.
This is because the register server does implicit type conversion to the type specified by the application.
The UAVCAN Specification does not require this behavior, though, so some simpler nodes (embedded systems in particular)
may just reject mis-typed requests.

If you restart the application now, you will see it use the updated PID gains.

Now let's try the linear regression service:

.. code-block:: sh

    yakut call 42 123:sirius_cyber_corp.PerformLinearLeastSquaresFit.1.0 'points: [{x: 10, y: 3}, {x: 20, y: 4}]'

The response should look like:

..  code-block:: yaml

    ---
    123:
      slope: 0.1
      y_intercept: 2.0

And the diagnostic subscriber we started in the beginning (type ``uavcan.diagnostic.Record``) should print a message.


Second node
-----------

To make this tutorial more hands-on, we are going to add another node and make it interoperate with the first one.
As the first node implements a basic thermostat, the second one simulates the plant whose temperature is
controlled by the thermostat.
Put the following into ``plant.py`` in the same directory:

.. literalinclude:: /../demo/plant.py
   :linenos:

You may launch it if you want, but you will notice that tinkering with registers by way of manual configuration
gets old fast.
The next section introduces a better way.


Orchestration
-------------

..  attention::

    Yakut Orchestrator is in the alpha stage.
    Breaking changes may be introduced between minor versions until Yakut v1.0 is released.
    Freeze the minor version number to avoid unexpected changes.

    Yakut Orchestrator does not support Windows at the moment.

Manual management of environment variables and node processes may work in simple setups, but it doesn't really scale.
Practical cyber-physical systems require a better way of managing UAVCAN networks that may simultaneously include
software nodes executed on the local or remote computers along with specialized bare-metal nodes running on
dedicated hardware.

One solution to this is Yakut Orchestrator --- an interpreter of a simple YAML-based domain-specific language
that allows one to define process groups and conveniently manage them as a single entity.
The language comes with a user-friendly syntax for managing UAVCAN registers.
Those familiar with ROS may find it somewhat similar to *roslaunch*.

The following orchestration file (orc-file) ``launch.orc.yaml`` does this:

- Compiles two DSDL namespaces: the standard ``uavcan`` and the custom ``sirius_cyber_corp``.
  If they are already compiled, this step is skipped.

- When compilation is done, the two applications are launched.

- Aside from the applications, a couple of diagnostic processes are started as well.
  A setpoint publisher will command the thermostat to drive the plant to the specified temperature.

The orchestrator runs everything concurrently, but *join statements* are used to enforce sequential execution as needed.
The first process to fail (that is, exit with a non-zero code) will bring down the entire *composition*.
*Predicate* scripts ``?=`` are allowed to fail though --- this is used to implement conditional execution.

The syntax allows the developer to define regular environment variables along with register names.
The latter are translated into environment variables when starting a process.

.. literalinclude:: /../demo/launch.orc.yaml
   :linenos:
   :language: yaml

Terminate the first node before continuing since it is now managed by the orchestration script we just wrote.
Ensure that the node script files are named ``demo_app.py`` and ``plant.py``,
otherwise the orchestrator won't find them.

The orc-file can be executed as ``yakut orc launch.orc.yaml``, or simply ``./launch.orc.yaml``
(use ``--verbose`` to see which environment variables are passed to each launched process).
Having started it, you should see roughly the following output appear in the terminal,
indicating that the thermostat is driving the plant towards the setpoint:

..  code-block:: yaml

    ---
    8184:
      _metadata_:
        timestamp: {system: 1614489567.052270, monotonic: 4864.397568}
        priority: optional
        transfer_id: 0
        source_node_id: 42
      timestamp: {microsecond: 1614489567047461}
      severity: {value: 2}
      text: 'root: Application started with PID gains: 0.100 0.000 0.000'

    {"2346":{"timestamp":{"microsecond":1614489568025004},"kelvin":300.0}}
    {"2346":{"timestamp":{"microsecond":1614489568524508},"kelvin":300.7312622070312}}
    {"2346":{"timestamp":{"microsecond":1614489569024634},"kelvin":301.4406433105469}}
    {"2346":{"timestamp":{"microsecond":1614489569526189},"kelvin":302.1288757324219}}

    # And so on. Notice how the temperature is rising slowly towards the setpoint at 450 K!

As an exercise, consider this:

- Run the same composition over CAN by changing the transport configuration registers at the top of the orc-file.
  The full set of transport-related registers is documented at :func:`pyuavcan.application.make_transport`.

- Implement saturation management by publishing the ``saturation`` flag over a dedicated subject
  and subscribing to it from the thermostat node.

- Use Wireshark (capture filter expression: ``(udp or igmp) and src net 127.9.0.0/16``)
  or candump (like ``candump -decaxta any``) to inspect the network exchange.
