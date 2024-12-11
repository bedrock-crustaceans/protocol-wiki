<script setup lang="ts">
import DefaultTheme from 'vitepress/theme';
import { useData } from "vitepress";
import { ref, watch, onMounted } from "vue";

interface GitHubAuthor {
  login: string;
  avatar_url: string;
  html_url: string;
}

const { Layout } = DefaultTheme;
const { frontmatter } = useData();

const contributors = ref([]);

async function getGitHubAuthor(login): Promise<GitHubAuthor | null> {

  const headers = {
    ...(!!import.meta.env.GITHUB_TOKEN && {
      Authorization: `Bearer ${import.meta.env.GITHUB_TOKEN}`,
    }),
  };

  try {
    const res = await fetch(`https://api.github.com/users/${login}`, { headers });
    if (res.ok) {
      return await res.json();
    } else {
      console.error(`Failed to fetch GitHub user: ${login}`);
      return null;
    }
  } catch (e) {
    console.error(e);
    return null;
  }
}

async function getContributors(): Promise<GitHubAuthor[] | null> {
  if (!Array.isArray(frontmatter.value.mentions)) return [];
  const contributorsList: GitHubAuthor[] = [];
  for (const login of frontmatter.value.mentions) {
    const user = await getGitHubAuthor(login);
    if (user) contributorsList.push(user);
  }
  return contributorsList;
};

watch(frontmatter, async () => {
  contributors.value = await getContributors();
});


onMounted(async () => {
  contributors.value = await getContributors();
});
</script>

<template>
  <Suspense>
    <Layout>
      <template v-if="frontmatter.Layout != 'home'" #doc-footer-before>
        <hr style="margin: 16px 0; border: none; border-top: 1px solid var(--vp-c-divider);">
        <h2 class="contributors" style="font-size: large; font-weight: 600;">Contributors</h2>
        <div v-if="contributors.length > 0" class="contributors">
          <a 
            v-for="contributor in contributors"
            :key="contributor.login"
            :href="contributor.html_url"
            :alt="contributor.login"
            target="_blank"
            rel="noopener noreferrer"
          >
            <img :src="contributor.avatar_url" :alt="contributor.login" :title="contributor.login" />
          </a>
        </div>
        <div v-else>No contributors found.</div>
      </template>
    </Layout>
  </Suspense>
</template>

<style lang="css">
.contributors {
  a {
    display: inline-block;
    background-color: var(--bg-color);
    border: var(--border);
    border-radius: 50%;
    overflow: hidden;
    line-height: 1;

    margin-inline: 0.2em -0.6em;
    transition-delay: 0.1s;
    transition:
      margin 0.2s,
      transform 0.2s;

    img {
      width: 2em;
      height: 2em;
      vertical-align: middle;
    }

    &:hover {
      transform: translateY(-0.3em);
    }
  }

  &:hover,
  &:focus-within {
    a {
      margin-right: 0.2em;
    }
  }
}
</style>