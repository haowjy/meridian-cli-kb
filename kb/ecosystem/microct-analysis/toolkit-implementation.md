# MicroCT Toolkit Notes

Watershed needs distinct markers: flood-filling seeds through connected, bridged bones can produce overlapping markers that cannot separate them. Small markers placed from visual evidence avoid that failure. Examining multiple planes can expose a separation hidden in one view.

Record the coordinate frame and transform with points: a slice or seed in an oriented volume cannot safely be interpreted as an original-volume coordinate without transforming it. See [measurement design](measurement-design.md) for orientation hazards and validation limits.
