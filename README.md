# Output Schema Extension

> **Experimental:** This extension currently has experimental status. Breaking changes are possible and community feedback is much appreciated. To learn more about the extension lifecycle, please refer to [A2A Extension and Protocol Binding Governance](https://a2a-protocol.org/latest/topics/extension-and-binding-governance/). This repository is staged under `vinoo999` pending TSC review and maintainer sponsorship.

This repository contains the specification for the **Output Schema Extension** for the Agent2Agent (A2A) protocol, URI `https://a2a-protocol.org/extensions/output-schema/v1`.

## Purpose

A client attaches a schema to a `SendMessage` request to describe the shape of the result it wants. The schema can be JSON Schema, XSD, a natural language description, or another declared schema language. The agent uses it as a generation hint and MAY report how it applied it. Format negotiation stays with `acceptedOutputModes`, and validation is left to SDKs and implementations.

## Specification

The full v1 specification can be found [here](./v1/spec.md).
