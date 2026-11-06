# Reference Resolution

Relative `-f` references use the logical task directory; `kb:` references use the authority's configured KB. These anchors remain separate when work happens in another checkout, so changing the task directory does not redirect KB lookup. A `kb:` reference must stay inside the KB root. See `src/meridian/lib/launch/reference.py` and `src/meridian/lib/launch/cwd.py` for resolution behavior.
