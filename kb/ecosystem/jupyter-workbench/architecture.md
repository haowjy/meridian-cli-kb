# Jupyter Workbench Architecture

CLI, MCP, and Python use the same service layer so notebook and session behavior does not diverge between interfaces. Typed results let each surface choose its own presentation. Domain packages interpret interaction events; the workbench keeps them generic.
