---
layout: post
title: The Tangent Variety to the Twisted Cubic
---

The twisted cubic.

Get the equation of the tangent variety to the twisted cubic, using M2.


{% raw %}
    i1 : R = QQ[X,Y,Z];
    
    i2 : S = QQ[s,t];

    i3 : f = map(S,R,matrix {{t+s, t^2 + 2*s*t, t^3 + 3*s*t^2}});
    
    o3 : RingMap S <--- R
    
    i4 : ker f
    
                 2 2     3      3             2
    o4 = ideal(3X Y  - 4X Z - 4Y  + 6X*Y*Z - Z )
    
    o4 : Ideal of R
{% endraw %}

