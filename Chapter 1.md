##1. Start a new application
###&nbsp;&nbsp;&nbsp;&nbsp;1.1 Git
###&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.1 Introduction
**Git** is a distributed VCS (Version Control System). If you are not using an VCS in your company, stop this tutorial right now and go to install one, we will talk later. I would recommand Git, a comparison of VCS tools or a detailed documentation about Git are out of the scope of this book. If you are already working with a different VCS (SVN, Mercurial or any other tool) that's perfectly fine, it will be trivial to translate the commands we will use in this tutorial to your preferred VCS syntax. We will be using basic operations like *get changes, update remote repository, compare changes*. If you are not using a VCS yet, this can be an opportunity to start with Git.

If you didn't yet, you can install Git from [here](http://git-scm.com/downloads).
Since the source code of this tutorial is hosted on [GitHub](https://github.com/symfony-tutorial/tutorial) all the remote references will be pointing to the GitHub repository. When following the instructions, always change
    `git@github.com:symfony-tutorial/tutorial.git` to `git@github.com:your_username/your_repository.git`.

If you didn't worked with github before, you may need to learn how to [set up an SSH key](https://help.github.com/articles/generating-ssh-keys/)




Enough **chattering**, let's get to the keyboard.

###&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.2 Your first repository
Go to your work directory and type in:
````bash
mkdir symfony-tutorial
cd symfony-tutorial
git init
touch composer.json
git add composer.json
git commit -m "first commit"
git remote add origin git@github.com:symfony-tutorial/tutorial.git
git push -u origin master
````




````php
<?php

namespace Blog\AppBundle\Controller;

use Blog\AppBundle\Entity\Category;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class DefaultController extends Controller
{

    public function indexAction($categorySlug = '', $postSlug = '')
    {
        $categoryRepository = $this->getDoctrine()->getRepository(Category::REPOSITORY);

        $categories = $categoryRepository->findAll();
        $menu = array();
        /**
         * @var Category $category
         */
        foreach ($categories as $category) {
            if (!count($category->getPosts())) {
                continue;
            }
            $menuItem = array(
                'label' => $category->getCategory(),
                'url' => $category->getSlug(),
                'active' => false
            );
            if ($category->getSlug() === $categorySlug) {
                $menuItem['active'] = true;
            }
            $menu[] = $menuItem;
        }
        return $this->render('BlogAppBundle:Blog:index.html.twig', array('menu' => $menu));
    }

    function renderArticle($articleId)
    {

    }

}

````
