---
title: "About"
permalink: "/about/"
layout: page
mathjax: true
---

These notes are on GitHub in [Jekyll][1] so I do not worry over a web server.

[1]: https://jekyllrb.com/

## Corrections

If you notice something wrong or outdated [please submit it as an issue][2]
(or even better as a pull request with the improvement you imagine).

[2]: https://github.com/AE1020/AE1020.github.io/issues

## License

Articles are licensed under "Creative Commons Attribution-NonCommercial 4.0".

Not that I particularly *want* people to copy posts and put them somewhere else
where I can't update them.  But bad actors would do it anyway regardless of the
license.  So if your use is well-intentioned, and you'd like to excerpt the
material then fine.  Just link back here, please.
 
## Theme Used

The theme is called ["Contrast"][3] and it is by [Niklaus Buschmann][4]. It was
simple and looked good. Also it comes with support for equations, which you can
enable with just `mathjax: true` in the page.

<script type="math/tex; mode=display">f^{(n)}(z) = \frac{n !}{2 \pi i} \int_{C}\! \frac{f(\zeta)}{(\zeta - z)^{n+1}} \mathrm{d}\zeta</script>

*(Note: An update to the version of Jekyll/Kramdown used by github.io broke
this feature a bit.  [I raised an issue on Contrast's GitHub.][5])* 

[3]: https://jekyllthemes.io/theme/contrast
[4]: https://github.com/niklasbuschmann
[5]: https://github.com/niklasbuschmann/contrast/issues/28

The instructions suggest forking the repo, but I just cloned it and then pushed
to an empty repository. That way I can search it in the web UI (since at time of
writing GitHub does not search forks).

I'll mention this uses the [Liquid][6] templating system so I can find the
documentation later when I need it.

[6]: https://shopify.dev/docs/themes/liquid/reference

{% comment %}
         1         2         3         4         5         6         7         8
12345678901234567890123456789012345678901234567890123456789012345678901234567890
{% endcomment %}
