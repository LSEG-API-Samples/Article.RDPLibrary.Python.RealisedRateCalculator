# Risk Free Rates and the LIBOR Transition - a Realised Rate Calculator

As you will be aware, the FCA is phasing out the LIBOR rate for various reasons and it is expected to  cease in the near future.
Market participants are expected to transition away from LIBOR to adopt alternatives - Risk Free Rates (RFR).

In the UK, for example. a reformed version of the Sterling Overnight Index (SONIA) is the recommended replacement as the preferred RFR for sterling Markets after the end of 2021. Other jurisdictions have alternative rates being proposed or developed - such as Secured Overnight Rate (SOFR) in the USA.

Refinitiv offers considerable resources and tools to help in the migration plan such a dedicated IBOR Transition App in our Eikon / Workspace desktop offerings, as well as other initiatives around Term Reference Rates, IBOR fallback language and Derived analytics.

From a developer's perspective the Refinitiv Data Platform APIs can also help in the migration process.

In this example I am going to implement a basic Realised Rate calculator using a couple of these APIs and the Refinitiv Data Platform library.