.. _architecture:

아키텍쳐
============

개요
--------

PyUAVCAN은 `UAVCAN 프로토콜 스택 <https://uavcan.org>`_ 의 전체 기능을 구현하였으며 GUI 소프트웨어, 진단 도구, 자동화 스크립트, 프로토타입, 여러 R&D 연구와 같은 어플리케이션에 적합하다.
**GNU/Linux**, **MS Windows**와 **macOS** 를 지원하도록 설계하였다.

이 문서를 이해하려면 UAVCAN의 기본에 대한 이해와 `asynchronous programming in Python <https://docs.python.org/3/library/asyncio.html>`_
를 이해하고 있어야 한다.

이 라이브러리는 여러 서브 모듈로 구성되어 있으며 각각은 해당 프로토콜을 잘 구분해서 구현되어 있다.:

- :mod:`pyuavcan.dsdl` --- DSDL 언어 지원: code 생성 및 객체 직렬화
  이 모듈은 `Nunavut <https://github.com/UAVCAN/nunavut/>`_ 위에 얇은 wrapper이다.

- :mod:`pyuavcan.transport` --- 추상 UAVCAN 트랜스포트 계층 모듈과 여러 구체적인 트랜스포트 구현(UAVCAN/CAN, UAVCAN/UDP, UAVCAN/serial 등)
  이 서브모듈은 관련 low-level API를 제공하며 데이터는 바이트의 직렬화된 블록으로 표현된다.
  사용자는 이 모듈을 기반으로 커스텀 트랜스포트 빌드도 가능하다.
  *일반적인 어플리케이션은 이 API를 직접 사용하는 경우는 드물다.*

- :mod:`pyuavcan.presentation` --- 이 계층은 DSDL 직렬화 로직으로 트랜스포트 계층을 바인드하며 상위-레벨 객체지향 API를 제공한다.
  이 계층에서 데이터는 자동 생성된 Python 클래스의 인스턴스로 표현된다.(코드 생성은 :mod:`pyuavcan.dsdl`가 관리)
  *일반적인 어플리케이션은 이 API를 직접 사용하는 경우는 드물다.*

- :mod:`pyuavcan.application` --- 어플리케이션을 위한 탑-레벨 API.
  factory :func:`pyuavcan.application.make_node`가 해당 라이브러리의 main entry 지점이다.

- :mod:`pyuavcan.util` --- 해당 라이브러리에서 사용하는 여러 유틸리티 함수와 클래스의 구조화된 집합. 유저 어플리케이션은 이를 이용하면 편리하다.

.. note::
   이 라이브러리를 사용하기 위해서 사용자는 :mod:`pyuavcan.application` API 문서와 :ref:`demo`를 훑어보도록 하자.

라이브러리의 전반적인 구조와 UAVCAN 프로토콜로 매핑은 다음 다이어그램을 참고하자.:

.. image:: /static/arch-non-redundant.svg

서브모듈의 의존 관계는 다음과 같다.:

.. graphviz::
    :caption: Submodule interdependency

    digraph submodule_interdependency {
        graph   [bgcolor=transparent];
        node    [shape=box, style=filled, fontname="monospace"];

        dsdl            [fillcolor="#FF88FF", label="pyuavcan.dsdl"];
        transport       [fillcolor="#FFF2CC", label="pyuavcan.transport"];
        presentation    [fillcolor="#D9EAD3", label="pyuavcan.presentation"];
        application     [fillcolor="#C9DAF8", label="pyuavcan.application"];
        util            [fillcolor="#D3D3D3", label="pyuavcan.util"];

        dsdl            -> util;
        transport       -> util;
        presentation    -> {dsdl transport util};
        application     -> {dsdl transport presentation util};
    }

어플리케이션 계층과 트랜스포트 구현 서브모듈을 제외하고 모든 서브모듈은 자동으로 import된다.  --- 이 두가지 모듈은 사용자가 명시적으로 import해줘야 한다.::

    >>> import pyuavcan
    >>> pyuavcan.dsdl.serialize         # OK, the DSDL submodule is auto-imported.
    <function serialize at ...>
    >>> pyuavcan.transport.can          # Not the transport-specific modules though.
    Traceback (most recent call last):
    ...
    AttributeError: module 'pyuavcan.transport' has no attribute 'can'
    >>> import pyuavcan.transport.can   # Import the necessary transports explicitly before use.
    >>> import pyuavcan.transport.serial
    >>> import pyuavcan.application     # Likewise the application layer -- it depends on DSDL generated classes.


트랜스포트 계층
---------------

UAVCAN 프로토콜 자체는 CAN bus (UAVCAN/CAN),
UDP/IP (UAVCAN/UDP), raw serial links (UAVCAN/serial) 등과 같은 다른 트랜스포트를 지원하기 위해서 설계되었다.
일반적으로 실시간 안전이 중요한 UAVCAN의 구현은 유효성 검증 노력을 줄이기 위해서 해당 프로토콜에서 정의한 트랜스포트의 일부 제한된 것들만 지원한다.
PyUAVCAN은 이런 점이 다르다. -- 신뢰성과 관련 깊은 임베디드 시스템보다는 어플리케이션 SW을 위해 만들어졌다;
즉 PyUAVCAN은 onboard를 비행체에 둘 수 없지만 엔지니어나 연구용으로 만든 컴퓨터에 넣어서 온보드 네트워크에서 구현, 이해, 유효성 검증, 진단을 하는데 도움을 얻을 수 있다.
따라서 PyUAVCAN은 확장성과 다양한 용도(어플리케이션에 적합)를 위해서 단순함과 제약 사항(임베디드에 적합)간에 교환이 발생한다.

라이브러리는 트랜스포트에 대한 지식없이도 가능한 코어로 구성되어 있다. 이 코어는 UAVCAN 프로토콜, DSDL 코드 생성, 객체 직렬화의 더 높은 레벨을 구현한다.
core는 추상 *transport model*를 정의한다. 이는 트랜스포트에 종속된 로직과 분리되어 있다.
추상 트랜스포트 모델의 주요 컴포넌트는 인터페이스 클래스 :class:`pyuavcan.transport.Transport`이며 동일 모듈 :mod:`pyuavcan.transport` 에서 여러 추가적인 정의를 동반할 수 있다.

라이브러리에서 트랜스포트 구현은 네스티드 서브모듈에 포함되어 있다.;
전체 목록은 다음과 같다.:

.. computron-injection::
   :filename: synth/transport_summary.py

..  important::

    Typical applications are not expected to initialize their transport manually, or to access this module at all.
    Initialization of low-level components is fully managed by :func:`pyuavcan.application.make_node`.

사용자는 :class:`pyuavcan.transport.Transport`의 서브클래스를 이용하여 커스텀 트랜스포트를 구현할 수 있다.

API 문서에서 *monotonic time*를 언급하는 경우, :meth:`asyncio.AbstractEventLoop.time`의 time 시스템을 의미한다.
asyncio 마다 기본적으로 :func:`time.monotonic`를 의미하며 이를 변경하는 것을 추천하지 않는다.
이런 원칙은 이 라이브러리의 다른 모든 컴포넌트에도 유효하다.


Media 서브-계층
++++++++++++++++


Typically, a given concrete transport implementation would need to support multiple different lower-level
communication mediums for the sake of application flexibility.
Such lower-level implementation details fall outside of the scope of the UAVCAN transport model entirely,
but they are relevant for this library as we want to encourage consistent design across the codebase.
Such lower-level modules are called *media sub-layers*.

Media sub-layer implementations should be located under the submodule called ``media``,
which in turn should be located under its parent transport's submodule, i.e., ``pyuavcan.transport.*.media.*``.
The media interface class should be ``pyuavcan.transport.*.media.Media``;
derived concrete implementations should be suffixed with ``*Media``, e.g., ``SocketCANMedia``.
Users may implement their custom media drivers for use with the transport by subclassing ``Media`` as well.

Take the CAN media sub-layer for example; it contains the following classes (among others):

- :class:`pyuavcan.transport.can.media.socketcan.SocketCANMedia`
- :class:`pyuavcan.transport.can.media.pythoncan.PythonCANMedia`

Media sub-layer modules should not be auto-imported. Instead, the user should import the required media sub-modules
manually as necessary.
This is important because sub-layers may have specific dependency requirements which are not guaranteed
to be satisfied in all deployments;
also, unnecessary submodules slow down package initialization and increase the memory footprint of the application,
not to mention possible software reliability issues.

Some transport implementations may be entirely monolithic, without a dedicated media sub-layer.
For example, see :class:`pyuavcan.transport.serial.SerialTransport`.


Redundant pseudo-transport
++++++++++++++++++++++++++

The pseudo-transport :class:`pyuavcan.transport.redundant.RedundantTransport` is used to operate with
UAVCAN networks built with redundant transports.
In order to initialize it, the application should first initialize each of the physical transports and then
supply them to the redundant pseudo-transport instance.
Afterwards, the configured instance is used with the upper layers of the protocol stack, as shown on the diagram.

.. image:: /static/arch-redundant.svg

The `UAVCAN Specification <https://uavcan.org/specification>`_ adds the following remark on redundant transports:

    Reassembly of transfers from redundant interfaces may be implemented either on the per-transport-frame level
    or on the per-transfer level.
    The former amounts to receiving individual transport frames from redundant interfaces which are then
    used for reassembly;
    it can be seen that this method requires that all transports in the redundant group use identical
    application-level MTU (i.e., same number of transfer pay-load bytes per frame).
    The latter can be implemented by treating each transport in the redundant group separately,
    so that each runs an independent transfer reassembly process, whose outputs are then deduplicated
    on the per-transfer level;
    this method may be more computationally complex but it provides greater flexibility.

Per this classification, PyUAVCAN implements *per-transfer* redundancy.


Advanced network diagnostics: sniffing/snooping, tracing, spoofing
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Packet capture (aka sniffing or snooping) and their further analysis (either real-time or postmortem)
are vital for advanced network diagnostics or debugging.
While existing general-purpose solutions like Wireshark, libpcap, npcap, SocketCAN, etc. are adequate for
low-level access, they are unsuitable for non-trivial use cases where comprehensive analysis is desired.

Certain scenarios require emission of spoofed traffic where some of its parameters are intentionally distorted
(like fake source address).
This may be useful for implementing complex end-to-end tests for UAVCAN-enabled equipment,
running HITL/SITL simulation, or validating devices for compliance against the UAVCAN Specification.

These capabilities are covered by the advanced network diagnostics API exposed by the transport layer:

- :meth:`pyuavcan.transport.Transport.begin_capture` ---
  **capturing** on a transport refers to monitoring low-level network events and packets exchanged over the
  network even if they neither originate nor terminate at the local node.

- :meth:`pyuavcan.transport.Transport.make_tracer` ---
  **tracing** refers to reconstructing high-level processes that transpire on the network from a sequence of
  captured low-level events.
  Tracing may take place in real-time (with PyUAVCAN connected to a live network) or offline
  (with events read from a black box recorder or from a log file).

- :meth:`pyuavcan.transport.Transport.spoof` ---
  **spoofing** refers to faking network transactions as if they were coming from a different node
  (possibly a non-existent one) or whose parameters are significantly altered (e.g., out-of-sequence transfer-ID).

These advanced capabilities exist alongside the main communication logic using a separate set of API entities
because their semantics are incompatible with regular applications.


Virtualization
++++++++++++++

Some transports support virtual interfaces that can be used for testing and experimentation
instead of physical connections.
For example, the UAVCAN/CAN transport supports virtual CAN buses via SocketCAN,
and the serial transport supports TCP/IP tunneling and local loopback mode.


DSDL support
------------

The DSDL support module :mod:`pyuavcan.dsdl` is used for automatic generation of Python
classes from DSDL type definitions.
The auto-generated classes have a high-level application-facing API and built-in auto-generated
serialization and deserialization routines.

By default, DSDL-generated packages are stored in the current working directory.
This is convenient because the packages contained in the same directory are importable by default.
If a different directory is used, it has to be added to the import lookup path manually
either via the ``PYTHONPATH`` environment variable or via :data:`sys.path`.

The main API entries are:

- :func:`pyuavcan.dsdl.compile` --- transcompiles a DSDL namespace into a Python package.

- :func:`pyuavcan.dsdl.serialize` and :func:`pyuavcan.dsdl.deserialize` --- serialize and deserialize
  an instance of an autogenerated class.

- :class:`pyuavcan.dsdl.CompositeObject` and :class:`pyuavcan.dsdl.ServiceObject` --- base classes for
  Python classes generated from DSDL type definitions; message types and service types, respectively.

- :func:`pyuavcan.dsdl.to_builtin` and :func:`pyuavcan.dsdl.update_from_builtin` --- used to convert
  a DSDL object instance to/from a simplified representation using only built-in types such as :class:`dict`,
  :class:`list`, :class:`int`, :class:`float`, :class:`str`, and so on. These can be used as an intermediate
  representation for conversion to/from JSON, YAML, and other commonly used serialization formats.


Presentation layer
------------------

The role of the presentation layer submodule :mod:`pyuavcan.presentation` is to provide a
high-level object-oriented interface and to route data between port instances
(publishers, subscribers, RPC-clients, and RPC-servers) and their transport sessions.

A typical application is not expected to access the presentation-layer API directly;
instead, it should rely on the higher-level API entities provided by :mod:`pyuavcan.application`.


Application layer
-----------------

Submodule :mod:`pyuavcan.application` provides the top-level API for the application and implements certain
standard application-layer functions defined by the UAVCAN Specification (chapter 5 *Application layer*).
The **main entry point of the library** is :func:`pyuavcan.application.make_node`.

This submodule requires the standard DSDL namespace ``uavcan`` to be compiled first (see :func:`pyuavcan.dsdl.compile`),
so it is not auto-imported.
A typical usage scenario is to either distribute compiled DSDL namespaces together with the application,
or to generate them lazily before importing this submodule.

Chapter :ref:`demo` contains a complete usage example.


High-level functions
++++++++++++++++++++

There are several submodules under this one that implement various application-layer functions of the protocol.
Here is the full list them:

.. computron-injection::
   :filename: synth/application_module_summary.py

Excepting some basic functions that are always initialized by default (like heartbeat or the register interface),
these modules are not auto-imported.


Utilities
---------

Submodule :mod:`pyuavcan.util` contains a loosely organized collection of minor utilities and helpers that are
used by the library and are also available for reuse by the application.
